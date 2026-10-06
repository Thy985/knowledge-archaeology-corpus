# OWASP Agent Memory Guard — Knowledge Archaeology

**定位**：OWASP 官方 Incubator 的运行时记忆防御中间件——插在 AI agent 与其 memory store 之间，用检测器管线 + 声明式策略 + 分类晋升图 + 完整性基线防护 **ASI06 Memory Poisoning**（及工具滥用/权限提升/过度自主）。

**考古日期**：2026-10-07（ARCH-2026-10-07-001）· **版本**：0.3.3 · **commit**：`8403091` · **skill**：knowledge-archaeology v3.2

## 产物清单
| 目录 | 内容 |
|---|---|
| `00_overview.md` | 项目定位、核心命题、关键事实、corpus 连接 |
| `01_project-layer/` | 项目地图（架构/模块/12 检测器/生命周期/配置/权威） |
| `02_engineering-knowledge/` | EK Graph（33 条 EK，六类边 links，每条 evidence 可回溯） |
| `03_knowledge-layer/` | 8 个 Generalized KO（aggregation_rule R1-R4 + 反例预算 + Epistemic） |
| `04_flow-atlas/` | 七类流（Control/State/Data/Evidence/Authority/Memory/Policy） |
| `05_candidates/` | 8 条未验证假说（C-01~C-08，与 KO 严格区分） |
| `06_validation/` | Validation 报告（Blind Reconstruction 9/9）+ 独立验证报告 + Reconciliation 记录 |
| `archaeology-runs/ARCH-2026-10-07-001/` | job_manifest / snapshot_artifact / run_metadata / independent_validation |

## 核心认知（一句话）
**记忆安全 = 分层防线**：内容检测（regex/ML）→ 语义分类（provenance 信任梯度）→ 完整性基线（SHA-256）→ 策略执行（block/redact/quarantine）→ 事件观测（SIEM/OTel）→ 可判定基准（AMSB）——且每层诚实声明边界（permissive 默认 / source_class 不认证 / digest 不验证 / 数字附语料）。

## 关键质量指标
- EK 33 条（游离 0）· KO 8 个（R1-R4 覆盖 100%）· 七类流 · Candidates 8
- 独立验证：CONFIRMED 10 / PARTIALLY_CONFIRMED 1 / DOWNGRADED 1 / MISSING 2
- AMSB 独立复跑：unguarded 15/15 breach（F）· strict 4/15（D，全 memory_persistence）· hardened 1/15（A）
- 关键争议：默认 strict 配置不含 persistence 检测器（官方对照设计明示）；92.5% recall 为自编语料非自适应对手度量

## 连接
- 上游：OWASP Top 10 for Agentic Applications ASI06（参考实现）· MITRE ATLAS AML.T0080.000
- Corpus：langgraph（10-06，secure_langgraph_memory 示例）/ owasp-agentic-skills（09-26）/ mem0 / letta-code（记忆实现侧）
