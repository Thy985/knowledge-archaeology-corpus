# Reconciliation — 独立验证修正合并

> 输入：`validation/independent_validation.md`（21 项判定）+ 自检。
> 原则：只合并修正（scope 收紧 / 事实纠错 / 层级降级 / 遗漏补录），不修改原始 Evidence（E-001~E-100 的行号与断言保留），不删除矛盾记录。

## R-01 持久化口径纠错（DOWNGRADED → 修正）

**原断言**：E-090、Snapshot 4.5、Data Model §3 称“13/13 provider 都是 JSON 文件、所有写入都是 json.dump 覆盖整文件”。

**独立证据**：
- SkillWeaver：`storage/skillweaver/skillweaver_generated_skills.py`（`skillweaver_provider.py:41`）。
- Dynamic-Cheatsheet：`global_cheatsheet.txt`（`dynamic_cheatsheet_provider.py:80-87`）。
- 其余 11 个 provider 使用 JSON。

**修正后**：`11/13 JSON + 1 .py + 1 .txt`；“写入是整文件覆盖”仅适用于 JSON 类，`.py`/`.txt` 也是整文件覆盖，但不是 `json.dump`。

**落地**：已在 `02_engineering_knowledge/evidence-index.md` E-090、`snapshot_artifact.md` §4.5、`01_project_layer/data_model_and_lifecycle.md` §3 修正。证据条目保留原 ID。

## R-02 baseline 口径冲突（NEEDS_HUMAN_REVIEW）

**冲突**：
- README：`11 baselines + 2 evolved = 13`（E-093）。
- Day5 扫描口径：`13 = 12 基线 + Evolver`。
- 代码：13 个枚举/映射，无 “evolved” 标记（E-003/E-004）。

**修正**：Corpus 中并列两套口径，标注 `NEEDS_HUMAN_REVIEW`；不作为任何 KO 的依据；C-05 保留。任何“基线 vs evolved”比较必须先由论文附录裁决。

## R-03 KO 层级降级（OVER_GENERALIZED）

| KO | 原层级 | 修正后 | 理由 |
|---|---|---|---|
| KO-06 | L4 Cognitive Model | **L3 Pattern**（scope: self-modifying code systems；Epistemic: Hypothesis/Pattern，单项目） | 仅 1 个项目、1 条链；无第二项目复现 |
| KO-07 | L4 Cognitive Model | **L3 Pattern**（scope: validator-in-the-loop；Epistemic: Pattern，单项目） | 同上；且“通过率抬升”未测量（C-03） |

落地：`03_knowledge_layer/ko.md` 标题与汇总表已改；`00_overview.md` 计数改为 L3×9 / L4×0。Cross-project 部分只保留在 `05_candidates/`（C-08~C-10）。

## R-04 补录独立发现的遗漏

新增两条 Candidates（不改 KO）：
- **C-11**：失败轨迹摄取语义不一致（异常任务进入各 provider，MemP revise / Agent-KB 丢弃 / others 可能写成功）。
- **C-12**：Evolver `_last_provided_cache` 为实例内存态，provider 每线程实例化，跨任务使用量可能永不更新 → 打分公式失效。

同时把 `take_in_memory` 失败只 warning、无重试，列入 `04_flow_atlas` 的 F1/F4 证据缺口说明与 C-13（新增）。

## R-05 阶段矩阵的 scope 收紧（PARTIALLY_CONFIRMED）

原表述“IN 阶段无返回”改为：**“源码只在 BEGIN 分支返回；IN 分支未见有效返回构造（部分实现可能返回空 MemoryResponse）”**。影响范围：EK-07、KO-02、Snapshot 4.x 的“7/13 失忆”措辞。
- 保留“能力缺失”的判断（依据分支位置）。
- 不保留“所有 IN 均为空对象”的绝对措辞（未逐行验证 7 个文件全部返回路径）。

## R-06 清理缺口补录

`PhaseValidator` 只在 `verdict=passed` 时清理隔离目录（`phase_validator.py:804-810`）；失败候选的副本会累积。列入 `01_project_layer/config_testing_governance.md` 与 C-14（可选）。

## 修正后的最终指标

| 指标 | 值 |
|---|---|
| Evidence | 100（S3×96，S0×4） |
| EK | 50（平均出度 2.68，游离 0） |
| KO | 9（**L3×9**；R1×2 / R2×2 / R4×5） |
| 反例 | 27（3/KO） |
| Candidates | 14（C-01~C-14） |
| Flow Edge | 68 |
| 独立判定 | CONFIRMED 8 / PARTIALLY 4 / DOWNGRADED 3 / CONTRADICTED 1 / NEEDS_HUMAN 1 / MISSING 2 / 合计 21 |
| 运行证据 | 0（S4/S5 缺口） |

## 未合并项（保留原样）
- “13 = 12 baseline + Evolver”：不合并为单一事实（见 R-02）。
- “Cerebra 工具代码可安全执行”：C-06 保持 Hypothesis。
- 论文数字：保持 S0-vendor。
