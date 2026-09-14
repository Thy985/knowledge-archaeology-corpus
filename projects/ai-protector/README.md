# AI Protector — Knowledge Archaeology

**定位**：AI agent 安全运行时——"Ship AI agents with guardrails — not prayers." 确定性防护层（No LLM in the loop · ~50ms · fully local · **provable**），以 Find → Protect → Prove 闭环落地。

**考古日期 / 版本**：2026-09-15 · v0.2.8（HEAD 8f57f85）· run ARCH-2026-09-15-001 · skill v3.2

**三选一实测第 3 点**：Aigis（09-13）→ Guardian（09-14）→ **AI Protector（本轮）**——防火墙品类"确定性安全网关"闭环数据点；AGT 留待四选实测第 4 点。

## 核心结论

- **Provable 是品类内独有能力**：Benchmark Hub（~5,070 场景，6 数据集归一化）+ 客观真值 grader（planted canary/secret，no LLM-as-judge，94%→99%）——"不可验证的分数不是证明"
- **双层防线**：proxy 7 层内容防护（Rules→Intent(A2 反混淆)→LLM Guard→Presidio→NeMo→Jailbreak ML→Harm ML）＋ agent RBAC 双门动作防护（pre/post-tool）
- **三态响应**：ALLOW/MODIFY/BLOCK（MODIFY=改写而非阻断，Aigis/Guardian 之外的第三态）
- **修复史**：NeMo 硬阻断→软贡献（误伤良性查询）；A2 variants-first -13pp 实验；冷启动 50s→0.9s；NeMo 低估依赖修复
- **⚠️ 绕过面（Auditor 发现）**：`/v1/chat/direct` 绕过端点**默认开启**（config.py:117 True，生产需显式关闭）；streaming 响应无输出过滤；管理端点（policies/rules）无认证

## 产物清单

| 层 | 文件 | 说明 |
|---|---|---|
| 00 Overview | [00_overview.md](00_overview.md) | 定位 / 关键数字 / 品类对照 / Top 5 发现 |
| 01 Project Layer | [01_project-layer/README.md](01_project-layer/README.md) | 项目地图（入口/模块/数据结构/配置/治理/依赖） |
| 02 Engineering Knowledge | [02_engineering-knowledge/README.md](02_engineering-knowledge/README.md) | 36+3 条 EK（EK Graph links 全覆盖，含 Reconciliation 增补） |
| 03 Knowledge Layer | [03_knowledge-layer/README.md](03_knowledge-layer/README.md) | 10 KO（R1-R4 聚合，L3×6/L4×3/L5×1） |
| 04 Flow Atlas | [04_flow-atlas/README.md](04_flow-atlas/README.md) | 七类流 + bypass flow 增补 |
| 05 Candidates | [05_candidates/README.md](05_candidates/README.md) | 9 条未验证假说 |
| 06 Validation | [06_validation/README.md](06_validation/README.md) | Truth/Coverage/Flow/Abstraction/反例/Epistemic + Reconciliation |
| Run Metadata | [archaeology-runs/ARCH-2026-09-15-001/run_metadata.yaml](archaeology-runs/ARCH-2026-09-15-001/run_metadata.yaml) | run_id/commit/skill_version/质量指标 |
| Snapshot | [archaeology-runs/ARCH-2026-09-15-001/snapshot_artifact.md](archaeology-runs/ARCH-2026-09-15-001/snapshot_artifact.md) | 仓库快照 + 项目基础地图 |
| Independent Validation | [archaeology-runs/ARCH-2026-09-15-001/independent_validation_report.md](archaeology-runs/ARCH-2026-09-15-001/independent_validation_report.md) | Auditor 盲重建报告（3 成功/3 错误/遗漏） |

## 质量指标

- 本地测试：**668 passed**（proxy 纯逻辑 203 + agent-demo 465）；400 项环境受限（无 Postgres/Redis）
- EK 39 条（links 全覆盖）· KO 10 个 · Candidates 9 条
- 基准数字（99%/91%/48ms/1900+）为项目自述 S7——本地未复现 benchmark，如实标注
- Validation：CONDITIONAL PASS（核心机制全代码级确认；绕过面已补录；2 项 NEEDS_HUMAN_REVIEW）
