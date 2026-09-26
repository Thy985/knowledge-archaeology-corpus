# 00 — Overview：A2A（Agent2Agent Protocol）

## 一句话定位
A2A 是一个**跨组织 AI Agent 协作的应用层协议规范**（非代码库）：用 proto3 定义"agent 如何发现彼此、协商交互、共享任务、交换状态"的抽象操作层与消息契约，并通过 JSON-RPC / gRPC / HTTP+JSON 三种标准绑定映射到传输。核心命题不是"如何实现一个 agent"，而是"**不同生态的 agent 之间如何以对等身份协作**"。

## 本次考古识别要点
- **仓库类型**：协议规范仓库（135 文件；核心资产 = `specification/a2a.proto` 812 行 + `docs/specification.md` 3618 行 + 1 条 ADR + 治理文档），**无运行时源码、无测试体系**（CI 11 个 workflow 全部为 lint/spelling/release/links 治理类）
- **核心认知资产（本次考古最有价值的 5 个发现）**：
  1. **Task 不可变性**（终态任务不可重启，一切后续工作 = 同 contextId 的新任务）——用"不可变任务 + 客户端管理 artifact 版本"换取可追溯性与实现简单性（协议级不变量）
  2. **Message-vs-Task 双轨**（无状态协商 vs 有状态执行；Hybrid Agent "先协商后承诺"）
  3. **核心冻结 + 扩展治理三明治**：扩展禁止改 core 数据结构/enum，只允许 metadata 叠加 + 新 RPC + 新状态，经两级治理门（experimental→official→core）晋升——核心稳定性铁律
  4. **安全不变量**：扩展/自定义绑定"不得绕过 agent 主认证与授权"；推送通知双侧安全（server 防 SSRF + client 验 JWT/JWKS/防重放）
  5. **ADR-001 ProtoJSON**：把 JSON 序列化规范性委托给 ProtoJSON 标准（标准化收益 vs enum SCREAMING_SNAKE_CASE 破坏性变更的可逆决策）

## 产物清单
| 文件 | 内容 |
|---|---|
| 01_project-layer/ | 项目地图（本快照，架构/抽象/状态/配置/治理/依赖） |
| 02_engineering-knowledge/ | 29 条 EK（EK Graph，六类边） |
| 03_knowledge-layer/ | 7 个 KO（aggregation_rule R1-R4）+ 五层阶梯递进 |
| 04_flow-atlas/ | 七类流（Control/State/Data/Evidence/Authority/Memory/Policy） |
| 05_candidates/ | 7 条未验证假说（含跨项目假说） |
| 06_validation/ | Blind Reconstruction + 判定统计 + 反例 + Reconciliation |

## 元数据
- run_id: `ARCH-2026-09-24-001`
- repository: `a2aproject/A2A` @ `43e0c874d3baba68ed84b98678d7f2268438e69f`
- 协议版本: v1.0.1（CHANGELOG 2026-05-26）
- skill_version: knowledge-archaeology v3.2（本 run 生效）
- 归档：corpus `projects/a2a/`（PR 待建）
