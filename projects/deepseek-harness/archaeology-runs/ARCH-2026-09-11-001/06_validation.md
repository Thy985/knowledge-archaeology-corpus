# 06 — Validation & Evidence（v0.1.5 REFRESH）

## 6.1 验证范围与方法

- **Truth**：全部新 EK/KO 声明附证据（notes 文件名 + 代码 symbol + 测试实测），可在 `repo/.agents/notes/implemented/…` 与 `packages/…` 回溯。
- **测试证据（S4，本轮实测）**：`python/sdk/tests` → **111 passed, 7 skipped**（Python 3.12.11 / pytest 3.35s）。首次运行失败记录：需 `pip install -e ./python/sdk` + `-e ./python/sdk-runtime`（PyPI 空壳 wheel 缺 `RUNTIME_MODE_ENV_VAR`）——已解决，非产品缺陷。
- **Blind Reconstruction**：写包前先独立读仓库（snapshot/git ls-tree/notes 抽样），再对照 09-03 基线，非"改基线文本"。

## 6.2 六类 Auditor 结论（独立 Auditor 报告详见 independent_validation_report.md）

| Auditor | 结论 |
|---|---|
| Truth | CONFIRMED（13/13 新 EK 证据可回溯；雷达声明与仓库证据分级标注） |
| Coverage | **09-03 Coverage Gap 确认并补偿**：.agents/notes（862→958）此前零引用；本轮补 13 EK。09-03 登记的 gaps（acp/code-runtime/feedback/web_rpc）部分补（feedback→EK-R11；code-runtime→EK-R02/R03；acp 仍浅） |
| Flow | PTC collapse 边、V3 envelope 状态转换、workspace-files 数据/权威边全部对应真实 symbol/notes |
| Abstraction | KO-09/KO-10 均为 L4（Cognitive Model）且标 Cross-project pending；无 L5 冒进；无"漂亮结论升层" |
| Counterexample | KO-09：collapse 前版本=反例（schema 省略但 executor 放行）；nested=true=合法例外。KO-10：09-05 containment 初版=反例（过窄）；CSP 禁网=反例（过严） |
| Epistemic | Fact/Observation/Hypothesis/Pattern/Model/Principle 分离；5 个新 Candidate 全部标 Hypothesis/Tentative/Scope-uncertain |

## 6.3 保留的 Contradictions 与 Counterexamples

- 09-03 保留项（compaction 有损 / SEARCH_FAILED 掩盖 / danger-full-access 绕过 + C-06/07/08）：**本轮未发现被修复或反证，保持开放**。
- 本轮新增矛盾记录：
  - 09-05 workspace-files 初版（containment）与 09-09 修订（继承 fs 授权）**互斥决策**——以 supersede 形式共存于 notes 历史，非错误，是设计演进证据。
  - 雷达"三模式联合训练"与仓库证据（仅 PTC 模式可核验）——不一致，按一方称处理。

## 6.4 质量指标（run_metadata.yaml 同步）

| 指标 | 值 |
|---|---|
| evidence_count | 45（32 基线核对 + 13 新 EK 证据） |
| engineering_knowledge_count | 45（32 存续 + 13 新增 EK-R01..R13） |
| generalized_ko_count | 10（8 存续 + 2 新增） |
| candidates_count | 14（9 基线 + 5 新增） |
| ek_avg_out_edges | ≥1（全部有出边） |
| isolated_ek_ratio | 0.0 |
| aggregation_rule_coverage | 1.0 |
| ko_avg_cluster_size | 5.0（KO-09:4 EK / KO-10:6 EK） |
| l4_ko_count | 9 |
| l3_ko_count | 1 |
| l5_ko_count | 0 |
| cross_project_pending_count | 5（KO-03/05/09/10 + C-R01..R04） |
| python_sdk_tests | 111 passed / 7 skipped（S4 实测） |

## 6.5 验证结论

**ACCEPT** —— 本轮为 REFRESH：基线 32 EK / 8 KO 在 v0.1.5 全部存续（无被反证项），新增 13 EK / 2 KO 有 notes+代码+测试三重证据；独立 Auditor 判定 CONFIRMED 为主（详见独立报告）。
