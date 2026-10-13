# ReviewBench 项目考古索引

**一句话定位**：GitHub 官方 AI 代码审查离线评测基准——以人工多源审校的 golden findings 为 ground truth，经 LLM 语义匹配→分类→评分三步管道，用 grounded/augmented precision-recall 可复现比较任意 review agent。

- **考古日期**: 2026-10-14
- **版本**: 1.0.0（commit `c17b05b`）
- **Skill Version**: knowledge-archaeology v3.2
- **run_id**: ARCH-2026-10-14-001
- **数据集**: corpus/manifest.json 219 PR + test set 25 + golden 219（full 4632 findings / 2623 TP / 56.6%）

## 产物清单

| 层 | 文件 | 内容 |
|----|------|------|
| 00 | `00_overview.md` | 一句话定位 + 考古元信息 + 核心发现预览 + 认知状态声明 |
| 01 | `01-project-layer/project_layer.md` | 模块地图/运行流程/核心数据结构/生命周期/配置/权限治理/依赖/边界 |
| 02 | `02-engineering-knowledge/ek_graph.md` | 28 EK（EK Graph 六类边 42 条，0 游离；含 Reconciliation 修正 R1-R3） |
| 03 | `03-knowledge-layer/knowledge_layer.md` | 8 KO（R1×3/R2×2/R3×2/R4×1 + 升维检查） |
| 04 | `04-flow-atlas/flow_atlas.md` | 七类流（Control/State/Data/Evidence/Authority/Memory/Policy） |
| 05 | `05-candidates/candidates.md` | 6 Candidates（C-01~C-06，全部 Hypothesis/未验证） |
| 06 | `06-validation/validation_report.md` | 六 Auditor + 盲重建 + 反例预算制 + §9 Reconciliation |
| 06 | `06-validation/independent_validation_report.md` | 独立 Auditor 盲重建（10 CONFIRMED/1 PARTIAL/2 MISSING） |
| runs | `archaeology-runs/ARCH-2026-10-14-001/` | job_manifest / snapshot_artifact / run_metadata / independent_validation_report |

## 关键认知（速览）

1. **评测可信度三不变量**（KO-05）：可复现（digest-pinned 镜像 + checkpoint 指纹隔离 + mirror 冻结快照）× 独立判定（golden 可删盘保密 + egress 白名单）× 防操纵（label 有效性检查 + 注入防护）——正式榜单评测必须三者齐备。
2. **被测者输出 = 不可信数据**（KO-02）：findings 文本转义 + 显式"data-not-instructions"，候选 agent 无法操纵自己的评分。
3. **匹配→计分去重**（KO-04）：covered 全算 recall、claimed 只认首个未认领 golden——防"一个发现认领多个金标刷 precision"。
4. **评测基准必须自证其偏**（KO-08）：golden 由 LLM 生产者主导（human 仅 1.5% TP 份额），METHODOLOGY §8 自列六类威胁；任何发布型基准都应附 Threats to Validity。

## Corpus 连接

- **AgentJudgeBench（PR #41）/ Transect（PR #42）**：评测可信度方法论三环同周考古（judge 审计层 / 结果可观测层 / 基准构造层）
- **deepseek-harness / agentevals / thinkingbox / dart-agent-core**：评测框架与评测方法论线
- **promptfoo**：validated 锚点（提示词评测对照）
- **opencode / hermes-agent / mcp**：Agent 运行时线（ReviewBench 是被测对象侧视角，正好对照）
