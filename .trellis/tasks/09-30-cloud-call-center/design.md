# 技术设计 — 云呼叫中心与电销系统 v0.1

> 依据 prd.md D1-D8 决策产出。设计粒度到"部署单元 + 模块边界 + 关键契约"，细节在 M1 开工前按需细化。

## 1. 总体架构：三个部署单元

```
┌────────────────────── biz-plane（业务域，JeecgBoot 单应用）──────────────────────┐
│ 平台域：用户/权限/租户/字典/报表/网关（JeecgBoot 原生 + 自研租户增强）              │
│ 业务域：客户/名单公海/产品/订单/物流/财务（核心交易手写，禁低代码生成）              │
└──────────────┬───────────────────────────────────────────────┬─────────────────┘
         Kafka 事件流（call.event.* / qa.task.*）          认证共享（JWT/用户数据）
┌──────────────┴───────────────────────────┐   ┌───────────────────┴────────────────┐
│ voice-plane（语音域，Java+Netty，独立部署） │   │ ai-plane（AI 域，独立部署）          │
│  call-control：ESL 网关+呼叫/坐席状态机    │   │  media-stream：FS 音频帧 WS 分发     │
│  dialer：策略可插拔外呼调度器              │   │  asr-gateway：FunASR 集群+云适配器   │
│  media-gateway：线路/号码/路由配置面        │   │  bot-engine：话术 DSL+LLM 兜底      │
│  compliance：黑名单/频控（强制前置）        │   │  qa-engine：离线转写+LLM 评分       │
│  FreeSWITCH 集群（WebRTC WSS / 录音 / SIP中继）│  │  tts-gateway：TTS 适配层        │
└──────────────────────────────────────────┘   └────────────────────────────────────┘
```

- **biz-plane**：JeecgBoot 单应用模式（Spring Cloud Alibaba 切换预案保留，v1 不启用）。
- **voice-plane**：与 biz-plane 零运行时耦合，仅经 Kafka 事件与内部 REST 交互；FreeSWITCH 节点独立扩容；Kamailio SIP 负载均衡在并发 >200 后引入。
- **ai-plane**：语音管道与对话引擎独立成单元，可单独发版；bot-engine 与 qa-engine 共享 asr/tts 网关。

## 2. 关键数据流

1. **呼入**：PSTN → SIP 中继 → FS → IVR/ACD 排队 → 坐席 WebRTC（WSS）。全程录音写对象存储，CDR 与通话事件（started/answered/bridged/ended）发 Kafka。
2. **机器人外呼**：dialer 产生呼叫 → FS 外拨 → `mod_audio_stream` 双声道音频帧 → asr-gateway（FunASR 流式）→ bot-engine（话术 DSL 优先，LLM 兜底，barge-in 检测）→ tts-gateway 回注 FS。触发转人工时呼叫注入 ACD 队列并携带对话上下文。
3. **弹屏**：call-control 事件 → Kafka → biz-plane WebSocket 推送坐席工作台，携带主叫号码 → 匹配客户档案。
4. **成交联动**：坐席弹屏下单 → 订单状态机（created→paid→shipped→signed→settled/after-sale）→ 物流聚合 API（快递100/快递鸟）→ 财务对账批处理。
5. **质检**：CDR 事件触发 qa-engine 拉录音 → 离线转写 → LLM 评分/违规检测 → 质检报告 + 申诉流。

## 3. 关键契约

| 契约 | 内容 |
|---|---|
| Kafka 事件 | topic `call.event.{tenantId}`，payload 含 tenantId/callId/caseId/agentId/时间戳；事件不可变，消费方幂等 |
| 内部 REST | voice-plane 暴露 `POST /calls`（发起外呼）、`POST /agents/{id}/actions`（坐席操作）；JWT 透传，网关层校验 |
| 话术 DSL | JSON：节点（开场白/分支/变量槽/转人工条件/打断策略）+ 版本化发布 + 试运行沙箱 |
| ASR 适配器 | 统一接口 `start(streamId)/onPartial/onFinal/stop`；实现：FunASR（自托管）、阿里/腾讯云（备选）；延迟指标按适配器上报 |
| 多租户规范 | 所有表含 `tenant_id` 且入复合索引；MyBatis-Plus 租户插件全局过滤；线路/号码/配额/计费为租户维度资源池 |

## 4. 技术取舍记录

- **业务域单体 vs 微服务**：10+ 人按三个部署单元并行已满足康威定律；全微服务的运维成本不划算。保留 JeecgBoot 微服务切换预案。
- **Shiro（随 JeecgBoot）**：换 Spring Security 会与上游冲突，保留 Shiro + 安全加固基线。
- **多租户实现**：应用层租户插件起步，预留 PG schema/RLS 演进；不做多库分片（v1 规模不需要）。
- **实时 ASR**：FunASR 自托管优先（成本/隐私/私有化一致），云 ASR 作为适配器实现存在，不动架构。
- **合规强制点**：compliance 模块在 dialer 策略之外独立成环——黑名单/频控/投诉名单作为外呼的强制前置过滤器，策略插件不可绕过。
- **预测式拨号**：实时统计滑动窗口（坐席空闲率/接通率/平均时长）→ 倍率决策；弃呼率护栏为硬约束，超限自动降速。

## 5. 部署与回滚

- **SaaS**：容器化编排（起步 docker-compose 多机，量后 K8s）；FS 节点有状态独立管理，RTP 端口段规划。
- **私有化**：docker-compose 拓扑（1 台全量 / 3 台分离 FS 与应用），安装升级脚本 + License 机器码绑定离线激活。
- **数据库**：Flyway 版本化迁移，每个里程碑的迁移必须带回滚脚本。
- **回滚**：三个部署单元版本独立，语音域可单独回滚不阻塞业务域发版。

## 6. 风险与未决项（非阻塞，设计阶段收口）

| 风险 | 应对 |
|---|---|
| 团队无 FreeSWITCH 调优经验（工期最大变数） | M1 前置 1-2 周技术预研：呼叫压测基线 + 回声/抖动调优脚本沉淀 |
| JeecgBoot 多租户深度不足 | M1 内完成租户 PoC（隔离/配额/线路池），不达标则自研租户模块 |
| ASR 端到端延迟 ≤1.5s 预算 | M3 管道先行压测，超预算则启用云 ASR 适配器并调整预算 |
| JimuReport/JimuBI 再分发条款 | 私有化交付物打包前核查，不合格替换为开源报表（如 UReport/积木开源版） |
| 合规线路资源未落实（商务前提） | 上线前必须落实线路商/运营商合同；系统侧 compliance 模块不依赖此进度先行开发 |
| 规模假设（未答复取默认） | 首期按 100 坐席/200 并发/日 5 万通设计，压测以此为目标基线 |
