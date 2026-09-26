# A2A（Agent2Agent Protocol）知识考古

**一句话定位**：跨组织 AI Agent 协作的应用层协议规范（proto3 定义 + 三标准绑定 JSON-RPC/gRPC/HTTP+JSON），2026-08-20 捐赠 Linux Foundation。核心命题：不同生态的 agent 如何以对等身份发现、协商、共享任务与交换状态。

## 考古信息
- **run_id**: ARCH-2026-09-24-001（每日 Knowledge Archaeology Runtime SOP）
- **commit**: `43e0c874d3baba68ed84b98678d7f2268438e69f`（main，2026-09-22）
- **协议版本**: v1.0.1（CHANGELOG 2026-05-26）
- **skill_version**: knowledge-archaeology v3.2
- **日期**: 2026-09-24

## 产物清单
| 目录/文件 | 内容 |
|---|---|
| 00_overview.md | 考古概览 + 5 个核心认知资产 |
| 01_project-layer/ | 项目地图（架构/抽象/状态机/配置/治理/依赖） |
| 02_engineering-knowledge/ | 29 条 EK（EK Graph，六类边，44 出边） |
| 03_knowledge-layer/ | 7 个 KO（R1-R4 聚合规则矩阵 + 五层阶梯） |
| 04_flow-atlas/ | 七类流（Control/State/Data/Evidence/Authority/Memory/Policy） |
| 05_candidates/ | 8 条未验证假说（含跨项目假说） |
| 06_validation/ | 盲重建验证（20 判定：16 CONFIRMED / 1 OVER_GENERALIZED / 2 MISSING / 0 CONTRADICTED） |
| archaeology-runs/ARCH-2026-09-24-001/ | 本 run 元数据 + job manifest + 仓库快照 |

## 关键发现（Top 5）
1. **Task 不可变性**：终态任务不可重启，一切后续工作 = 同 contextId 的新任务（可追踪性由"不可变任务 + 客户端持有版本"换取）
2. **Message-vs-Task 双轨**：无状态协商 vs 有状态执行（"先协商后承诺"）
3. **核心冻结 + 扩展治理三明治**：扩展禁止改 core 类型，只允许 metadata 叠加 + 新 RPC + 新状态，经两级治理门晋升
4. **安全不变量**：扩展/绑定"不得绕过主安全控制"；推送通知双侧安全（防 SSRF + JWT/JWKS/防重放）
5. **ADR-001 ProtoJSON**：JSON 序列化规范性委托 ProtoJSON（可逆决策，标准化收益 vs enum 大小写破坏性变更）

## 验证结论
PASS_WITH_NOTES：无 CONTRADICTED；1 项快照笔误（RPC 计数）经 Reconciliation 修正；2 项覆盖缺口（错误映射/未精读文档）记入 refresh_pending。
