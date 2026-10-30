# Agent Substrate — Knowledge Archaeology

**一句话定位**：Google 开源的 **Kubernetes-native 零信任 agent 运行时**——大量有状态 actor 映射到小池 worker，空闲快照出内存、请求到达恢复（10x 密度 / <500ms resume / >500 activations/sec 为文档目标），gVisor/microVM 双沙箱、沙箱内零凭据、egress 网关外注凭据、默认拒绝网络。

## 考古索引
| 产物 | 路径 | 说明 |
|---|---|---|
| Overview | `00_overview.md` | 项目定位 + 认知增量 + 质量指标 |
| Project Layer | `01_project-layer/` | 架构/模块/生命周期状态机/配置/依赖/治理 |
| Engineering Knowledge | `02_engineering-knowledge/` | EK Graph（41 条，6 类边，含 .agents/skills 治理技能层） |
| Knowledge Layer | `03_knowledge-layer/` | 7 个 Generalized KO（R1-R4 聚合，可回溯 EK） |
| Flow Atlas | `04_flow-atlas/` | 七类流（Control/State/Data/Evidence/Authority/Memory/Policy） |
| Candidates | `05_candidates/` | 5 个 Hypothesis / Observation（含跨项目假说） |
| Validation | `06_validation/` | 六审计 + Independent Validation Report + Reconciliation |
| Run 归档 | `archaeology-runs/ARCH-2026-10-31-001/` | 本次 run 原始产物（不覆盖历史） |

## 考古日期 / 版本
- 日期：2026-10-31
- repository：`agent-substrate/substrate` @ `542eca1e348e80001dcca48789e9433a84eeb214`（main，2026-10-30）
- Skill：knowledge-archaeology v3.2（无 Skill Improvement Proposal 产出）
- 交付方式：corpus push + PR（飞书落盘未授权，默认不做）
