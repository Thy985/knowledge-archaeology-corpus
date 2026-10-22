# strands-box — Knowledge Archaeology

**定位**: AWS 官方开源（Apache-2.0）OS 级 AI agent 行为遏制沙箱引擎（v0.1.0，Rust）——containment（macOS Seatbelt/Linux namespace）+ Dogwood 语义策略（box.toml + policy.dw），harness agnostic，决策默认拒绝。

**考古日期/版本**: ARCH-2026-10-23-001 · skill v3.2 · HEAD `8c46dac`（2026-10-22）
**status**: COMPLETED（initial）

## 认知主线（见 00_overview / 03_knowledge-layer）
1. fail-closed 是逐层机制而非顶层口号（四拒绝语义 + 引擎安全 pin + 类型化路径 + 网关冲突拒绝）
2. 执行面与决策面物理分离（三 PEP 共享唯一 PDP，enforcement 在 agent 进程外）
3. 机密性边界 = 边界持有（凭证 Vault 出站注入，FORBIDDEN_PATHS 永不授予）
4. 时间性策略 + 统一事件语义（dogwood-local-engine durable history + fs:read/http:request 归一）
5. 治理即接口冻结（interface-frozen + decisions.md ADR 降格 + 测试钉死行为）

## 产物清单
| 目录 | 内容 |
|---|---|
| 01_project-layer/ | 项目地图 |
| 02_engineering-knowledge/ | EK Graph（B01~B21，Reconciliation 后 21 条） |
| 03_knowledge-layer/ | KO-B01~B05（R1×1 R2×1 R3×2 R4×1） |
| 04_flow-atlas/ | 七类流 |
| 05_candidates/ | C-B01~B07（Hypothesis / Cross-project pending） |
| 06_validation/ | Validation + Independent Validation（11 CONFIRMED/1 PARTIALLY/0 CONTRADICTED）+ Reconciliation（R-1~R-5） |
| archaeology-runs/ARCH-2026-10-23-001/ | run_metadata / job-manifest / snapshot |

## 跨项目连接
- **dogwood**（策略语言本体，09-08 已考古）——Box 是其运行时载体（dogwood-local-engine =1.0.0 精确 pin）
- **MXC**（10-19）/ **agent-aegis**（10-22）/ **deepseek-harness**（validated）——治理位置谱系：OS 遏制（Box/MXC）→ 进程边界（dsh）→ 进程内插桩（aegis）
- **SIP**: 无（本轮未产生 Skill Improvement Proposal）
