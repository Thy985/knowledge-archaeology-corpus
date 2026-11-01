# agent-aegis — Knowledge Archaeology

**定位**: AI agent 的治理层运行时（Acacian/aegis，v1.0.0，Python/MIT）——"One API, 12 frameworks, every governance primitive"；进程内插桩式运行时防御 + 声明式 YAML 策略引擎 + 审批门 + Merkle 防篡改审计 + AGEF/AGP 自述标准。

**考古日期/版本**: ARCH-2026-10-22-001 · skill v3.2 · HEAD `134580e`（2026-08-29）
**status**: COMPLETED（initial）

## 认知主线（见 00_overview / 03_knowledge-layer）
1. 声明 vs 评估分离（justification_gap 为审批核心判据）
2. 策略即数据 + 防遮蔽缓存（条件规则 action 不缓存）
3. 信任委托单调约束（trust 沿委托链非递增）
4. 零代码接入 vs 完整 API 并存（env / auto_instrument / Aegis.init 三梯度）
5. 自述标准分层标注（AGEF/AGP Draft ≠ 生态采纳）

## 产物清单
| 目录 | 内容 |
|---|---|
| 01_project-layer/ | 项目地图 |
| 02_engineering-knowledge/ | EK Graph（G01~G19，Reconciliation 后 19 条） |
| 03_knowledge-layer/ | KO-G01~G05（R1×2 R2×1 R3×1 R4×1） |
| 04_flow-atlas/ | 七类流 |
| 05_candidates/ | C-G01~G07（Hypothesis / Cross-project pending） |
| 06_validation/ | Validation + Independent Validation（11 CONFIRMED/1 PARTIALLY/0 CONTRADICTED）+ Reconciliation（R-1~R-5） |
| archaeology-runs/ARCH-2026-10-22-001/ | run_metadata / job-manifest / snapshot |

## 跨项目连接
- aigis / guardian / ai-protector（运行时防御同品类）/ agent-governance-toolkit（无状态策略引擎对照）/ archon（10-21，控制面 gate vs 插桩治理）/ deepseek-harness（approval seam）/ mcp（MCP 信任边界）
- **SIP**: 无（本轮未产生 Skill Improvement Proposal）
