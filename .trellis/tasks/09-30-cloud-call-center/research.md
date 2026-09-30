# 调研记录 — 开源选型（2026-09-30）

> 本文件是 prd.md Research Notes 的落地归档，供实现/检查阶段引用。

## 开源平台盘点结论

**没有任何现成开源平台能同时满足本任务的六项决策**（SaaS+私有化、AI 全量、四种外呼、Java、完整交易套件）。

| 项目 | 技术栈 | 结论 |
|---|---|---|
| FreeIPCC (github.com/lihaiya/freeipcc) | Java + Python + FreeSWITCH | 商业授权"伪开源"（付费购源码方可商用，非 OSI 许可），124 star，多租户不明——仅作参考实现，不构成二开路线 |
| VICIdial / GOautodial | Perl/PHP + Asterisk | GPL；预测式拨号逻辑可参考；栈老旧无多租户无 AI，不直接二开 |
| FusionPBX / FreePBX | PHP | PBX 管理面，非呼叫中心业务系统 |
| Jambonz (jambonz.org) | drachtio + rtpengine (Node) | 开源语音 AI 编排平台：SIP 原生、STT/TTS/LLM 可插拔、自托管——AI 管道架构参考与候选组件 |
| LiveKit Agents / Pipecat | Python | 语音 AI 对话框架（STT→LLM→TTS、barge-in）——机器人管道参考 |
| 若依 / 悟空CRM / mall | Java | 管理脚手架与 CRM/电商模块设计参考 |

## JeecgBoot 核查（2026-09-30，github.com/jeecgboot/JeecgBoot）

- 许可证 **Apache-2.0**（徽章、声明与社区确认；"勿冒充官方品牌"为非强制社区倡议）→ SaaS + 私有化交付无授权障碍。
- 48k star / 16.2k fork，最新 v3.9.5（2026-08-27），十年迭代高活跃。
- 栈：Spring Boot 4.1 + JDK 17/21/24 + MyBatis-Plus 3.5 + Shiro 3 + langchain4j 1.17；前端 Vue3 + TS + Vite + Ant Design Vue 4；单应用与 Spring Cloud Alibaba 微服务模式可切换；数据库 MySQL/PG/Oracle/达梦/人大金仓/TiDB。
- 已接受前提：开源版多租户深度不足（需自研隔离/配额/计费并 PoC）；JimuReport/JimuBI 为 freemium 闭源组件（私有化再分发条款须核查，不合格替换）；上线前安全加固基线（历史漏洞类别：注入/默认密钥/管理端点暴露）；fork 后锁定版本不跟上游。
- 定位约束：仅承载平台域与业务域管理面；交易域核心手写（禁用表单代码生成）；语音域与 Jeecg 运行时零耦合。

## 其他组件

- FunASR：自托管中文实时 ASR 首选（成本/隐私/私有化一致性），云 ASR 做适配器备选。
- Kamailio：SIP 负载均衡（M2 后按并发引入）。
- mod_audio_stream 类方案：FreeSWITCH 实时媒体外送管道（AI 域输入）。
- 授权红线：GPL/AGPL 组件进入 SaaS 与私有化交付前逐一核查传染性。

## 参考链接

- https://jambonz.org/
- https://github.com/jeecgboot/JeecgBoot
- https://github.com/lihaiya/freeipcc
- https://github.com/signalwire/freeswitch
