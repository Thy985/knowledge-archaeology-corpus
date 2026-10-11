# Independent Validation Report — AgentJudgeBench（ARCH-2026-10-12-001）

> 独立 Auditor 盲重建：**不读主考古产物**，独立重读仓库建立 Independent Findings 后对比。
> 盲审计读证路径（独立于主考古）：tests/ 全树、`sygra/core/execution_callbacks.py`、`sygra/core/eval/`、`sygra/core/graph/nodes/llm_node.py`、`base_task_executor.py:990-1060`（输出写入）、`graph_config_corrupted_gt.yaml`（corrupted 引用）、`tasks/agentic_bfcl_judge_eval/analyze_results.py`（glob/parts/argparse 全文）、Makefile、`.github/workflows/`。

## 一、Independent Findings（独立重建，先于对比）

**IF-1 输出文件结构（代码实证）**：`base_task_executor.py:1001-1045`——`out_file = output_dir + "/{run_name_prefix}output{ts_suffix}.jsonl"`；output_dir=run_judge patch 的 `<task_dir>/<generator>/judged_by_<judge>`；run_name 默认 `judge_<gen>_<judge>_<diff>`（run_judge.py:56-57）；ts 默认追加（--output_with_ts True）。**实际输出 = 3 层单文件**：`<generator>/judged_by_<judge>/judge_*_output_*.jsonl`。

**IF-2 analyze 匹配协议（代码实证）**：`analyze_results.py:51-52`——glob `"*/judged_by_*/*/*.jsonl"`（**4 层**），`len(parts) < 4 → continue`；condition/difficulty 从 `parts[2]`（run_dir **目录名**）经正则推断（parse_run_dir）。

**IF-3 不兼容结论**：IF-1 输出 3 层（run_name/ts 在**文件名**）；IF-2 要求 4 层（run_name 在**目录名**）。**默认复现流程下 analyze_results.py 匹配不到任何输出文件**。README 内部亦不一致（Step 3 声称 `judged_by_<judge>/<difficulty>.jsonl` 3 层；Step 8 声称 walks `<run_name>/*.jsonl` 4 层）——但代码是权威：BaseTaskExecutor 不建 run_name 子目录。

**IF-4 eval 子系统存在**：`sygra/core/eval/metrics/`——unit_metrics（exact_match/action_within_bbox/scroll_amount/scroll_direction/typed_value_match + registry）+ aggregator_metrics（accuracy/f1/pass_at_k/pass_power_k/precision/recall/step_efficiency + registry）。**ABB 任务包未 import 该子系统**（grep tasks/agentic_bfcl_judge_eval/*.py 无 `from sygra.core.eval`）——ProgrammaticJudge 是 task 内嵌 lambda，未走指标注册表。

**IF-5 C3 数据语义**：`tasks/agentic_bfcl_judge_eval/prepare_corrupted_gt.py` 的新 record = `dict(rec)` 复制 + 追加 `true_expected_responses` / `corrupted_expected_responses` 两字段——**原 `expected_responses` 字段保留真值**；corrupted config 的 with-GT 节点 prompt 用 `{corrupted_expected_responses}`（line 159），程序化 judge 读 `state["expected_responses"]`（真 GT）。`true_expected_responses` 字段为备查冗余。

**IF-6 LLMNode 契约**：`llm_node.py`——`output_keys` 为 list 时 assert 必须有 `post_process`。JudgeToolCallsPostProcessor 的存在是图运行时强制契约，非可选装饰。

**IF-7 其余核实**：nogt_05 与 main 仅 2 处 prompt diff（155/228 行，"default 1.0"→"default 0.5 when uncertain"）✓；GPT-5 温度 hack（run_generate.py pop temperature）✓；难度拆分（DIFFICULTY_MAP + KEEP_FIELDS）✓；derangement（按 dag_type 分组、无自映射循环、singleton 自交换报告）✓。

## 二、与主考古逐项对比（判定）

| 主考古项 | 判定 | 依据 |
|---|---|---|
| EK-01 四维符号打分 | **CONFIRMED** | IF-7 + ProgrammaticJudge 全文独立重读 |
| EK-02 alignment 公式 | **CONFIRMED** | ComputeLLMVsProgrammaticAccuracy 重读 |
| EK-03 双分支同图 | **CONFIRMED** | graph_config.yaml 两节点 + edges |
| EK-04 answer-only 提示 | **CONFIRMED** | 提示全文（"plausibly helps"） |
| EK-05 C3 derangement | **CONFIRMED**（措辞微调） | IF-5：程序化 judge 读 `expected_responses`（非 `true_expected_responses` 字段）——语义一致，字段措辞修正 |
| EK-06 难度三档 | **CONFIRMED** | IF-7 |
| EK-07 保守默认 | **CONFIRMED** | prompt 全文 |
| EK-08 safe_parse | **CONFIRMED** | think/fence 剥离重读 |
| EK-09 Serialize 预处理器 | **CONFIRMED** | 四字段序列化重读 |
| EK-10 C2 0.5-default | **CONFIRMED** | IF-7（diff 仅 2 处） |
| EK-11 freeform | **CONFIRMED** | 最小 prompt 重读 |
| EK-12 图运行时 | **PARTIALLY_CONFIRMED** | IF-4/IF-6：图运行时描述真；**遗漏 LLMNode 契约**与宿主 eval/metrics 并行能力（补 Observation，不改 EK 结论） |
| EK-13 参数 patch 模式 | **CONFIRMED** | run_judge.py:51-54 重读 |
| EK-14 GPT-5 hack | **CONFIRMED** | IF-7 |
| **EK-15 analyze 条件推断** | **PARTIALLY_CONFIRMED（实质缺陷遗漏）** | IF-1/2/3：正则逻辑真，但 **glob 4 层 vs 实际 3 层不兼容**——主考古未标注"默认流程 analyze 匹配不到输出"的管道缺陷 |
| KO-01 程序化锚定 | **CONFIRMED** | 证据链完整 |
| KO-02 提示校准器 | **PARTIALLY_CONFIRMED** | 单项目经验升 L4：必要非充分已声明；**建议标 Cross-project pending**（无第二项目证据） |
| KO-03 无 GT 降级 | **CONFIRMED**（已标 Hypothesis） | 单实例标注正确 |
| KO-04 GT 污染探针 | **CONFIRMED** | IF-5 验证探针/被探针隔离 |
| KO-05 基础设施分层 | **PARTIALLY_CONFIRMED** | 分层成立（宿主项目支撑）；"条件靠命名约定治理"维度受 IF-3 削弱（约定有缺陷） |
| C-01~C-06 | **CONFIRMED** | 候选边界合理 |
| **遗漏（MISSING）** | eval/metrics 子系统（01 层语境）；**analyze 管道 glob 缺陷（新 C-07）** | IF-4 / IF-1/2/3 |
| Contradictions | 0 | 主考古无自相矛盾 |
| Over-generalization | 0 | 未发现跨项目冒充 |

## 三、判定统计

| 判定 | 数量 | 条目 |
|---|---|---|
| CONFIRMED | 15 | EK-01/03/04/05/06/07/08/09/10/11/13/14, KO-01/03/04, C-01~C-06（按项计） |
| PARTIALLY_CONFIRMED | 3 | EK-12, EK-15, KO-02, KO-05（4 项——EK-12 与 KO-05 相关） |
| DOWNGRADED | 0 | — |
| OVER_GENERALIZED | 0 | — |
| MISSING | 2 | eval/metrics 语境（01）、analyze glob 缺陷（C-07） |
| CONTRADICTED | 0 | — |
| NEEDS_HUMAN_REVIEW | 0 | — |

## 四、3 个成功 + 3 个错误（盲审计自省）

**成功**：
1. IF-3：识别 analyze glob 与实际输出结构不兼容——主考古把 analyze 当可用管道末端，未标注缺陷
2. IF-4：识别宿主 eval/metrics 指标注册表子系统并确认 ABB 未使用（覆盖边界清晰化）
3. IF-6：LLMNode 的 post_process 强制契约解释了 processor 存在必要性（主考古未明说）

**错误**（盲审计自身修正，非主考古）：
1. 初始假设"README 输出 = 3 层 `<difficulty>.jsonl`"→ 实测文件名含 run_name+ts（`judge_*_output_*.jsonl`）——README 简化描述误导，以代码为准
2. 初始假设"analyze 的 run_dir 在目录层"→ 确认 parts[2] 是目录但实际不存在该目录——方向对但需代码验证
3. 初始未注意到 `true_expected_responses` 与程序化 judge 实际读取键（`expected_responses`）的差异——IF-5 修正

## 五、Benchmark case（以主考古 KO-02 为锚）

- **独立重建**：仅凭仓库 prompt diff（nogt_05 vs main 两处）+ freeform 变体，能否独立得出"提示校准器"结论？→ 能：diff 明确显示仅提示措辞变化改变评分默认值，无代码逻辑差异，提示结构与对齐度存在工程关联（**可独立重建，CONFIRMED 机制**）。
- **升维检验**：能否从本项目推出通用 L4 "提示词是校准器"？→ 不能（单项目、无第二任务对照）→ **支持主考古"必要非充分"自反例，但 L4 需 Cross-project pending 标注**（Reconciliation 采用）。

## 六、Reconciliation 输入（阶段⑥采用，禁改原产物原则：仅并入 corpus 最终版）

1. EK-15 补缺陷标注：analyze glob `*/judged_by_*/*/*.jsonl`（4 层）与 BaseTaskExecutor 实际输出（3 层，run_name 在文件名）不兼容 → 默认复现流程 analyze 匹配不到文件（需手动整理或修 glob）。
2. 04 Flow Atlas（Data Flow）修正：run_judge 实际输出 `<generator>/judged_by_<judge>/judge_*_output_<ts>.jsonl`（3 层）；analyze 期望 4 层目录结构 → 标注不匹配边。
3. 05 新增 C-07：analyze 管道 glob 缺陷（复现链需人工整理或脚本修正）。
4. 01 补 Observation：宿主项目含 eval/metrics 指标注册表（unit/aggregator），ABB 未使用（task 内嵌 lambda 路径）。
5. KO-02 加注 Cross-project validation pending（L4 单项目证据）。
6. EK-05 措辞微调：程序化 judge 实际读 `expected_responses`（prepare 保留真值），`true_expected_responses` 为备查冗余字段。
