# 05 Candidates — AgentJudgeBench（ARCH-2026-10-12-001）

> 未确认内容留在候选。Hypothesis 不冒充 Fact；Cross-project Candidate 不写成已验证 Principle。

---

## C-01 程序化 judge 本身的正确性边界（Hypothesis）
- **内容**：ProgrammaticJudge 四维指标对"参数语义等价"不敏感（如 0.75 vs 0.005 视为结构错误之外的等价，仅检查参数名），因此对齐度高只证明"LLM judge ≈ 符号公式"，不证明 judge 对人类标注正确。
- **证据**：`task_executor.py` ProgrammaticJudge docstring（"No value correctness checks"）；graph_config.yaml prompt（"Assume the programmatic judge is the ground truth"）。
- **缺失证据**：仓库无人类标注对比实验（需人工抽查子集）。
- **验证路径**：抽 50-100 条记录做人类 vs 程序化 judge 一致性测量。

## C-02 derangement 语义相关性未控（Hypothesis）
- **内容**：C3 corrupted GT 从同 dag_type 记录随机取（derangement），但相邻记录的期望工具集可能语义相似，corrupted GT 可能"碰巧"部分正确，稀释 anchoring 效应测量。
- **证据**：`tasks/agentic_bfcl_judge_eval/prepare_corrupted_gt.py`（按 dag_type 分组随机排列，无相似度过滤）。
- **缺失证据**：corrupted GT 与实际 GT 的语义距离分布未统计。
- **验证路径**：计算 corrupted vs true expected 的工具集 IoU 分布；按 IoU 分层重测对齐度。

## C-03 0.5-default 的跨任务迁移性（Cross-project Candidate）
- **内容**：C2（without-GT 0.5-default）对齐度变化是"标度敏感"而非"judge 能力提升"——该结论在本项目成立，但对其他评测任务（如自由文本打分）是否成立未验证。
- **证据**：`graph_config_nogt_05.yaml`（仅 prompt 两处 diff）+ 论文 Table 3 主张。
- **缺失证据**：无第二个任务的 0.5-default 对照实验。
- **验证路径**：在 promptfoo 断言 / 引擎 evaluation 层复跑同类变体。

## C-04 freeform vs 结构化 prompt 的鲁棒性（Hypothesis）
- **内容**：RQ6 freeform 提示对齐度差异可能对模型族敏感（20B 小模型更依赖结构化 rubric）。
- **证据**：`graph_config_freeform.yaml`（最小 prompt）+ 论文 Fig 7。
- **缺失证据**：仓库内无按 judge 规模分层的 freeform 分析脚本（需对 analyze CSV 二次过滤）。
- **验证路径**：analyze_results.py 输出按 judge 族分层统计。

## C-05 分支与 main 的漂移（Observation，需 refresh）
- **内容**：本分支为 SyGra 派生 scratch 分支（commit 级快照 2026-05-06）；上游 main 后续演化（synthetic data pipeline 主线的修复/重构）是否影响 AgentJudgeBench 复现未测。
- **证据**：分支名 `scratch/agent_judge_bench` + depth 1 克隆限制。
- **缺失证据**：git 历史 diff（需完整克隆）。
- **验证路径**：完整克隆对比 main 与 scratch 分支中 sygra/ 骨架差异。

## C-06 judge 规模效应的跨条件稳定性（Cross-project Candidate）
- **内容**：6 judge 从 20B（GPT-OSS-20B/QwQ-32B）到 frontier（GPT-5.4），论文主张对齐随规模提升——但该梯度是否在所有难度/GT 条件下单调未在仓库脚本中硬编码验证（依赖实验结果 CSV）。
- **证据**：README judge 列表 + RQ5（judge scale）复用 main 图。
- **缺失证据**：仓库无结果 CSV（数据集在 HF，结果需复跑）。
- **验证路径**：复跑 C1 网格后按 judge 规模排序检验单调性。

## C-07 analyze 管道 glob 缺陷（Observation，独立审计确认）
- **内容**：`tasks/agentic_bfcl_judge_eval/analyze_results.py` 的 glob `*/judged_by_*/*/*.jsonl` 要求 4 层目录结构（run_name 作目录名，condition/difficulty 从 `parts[2]` 目录名正则推断）；而 `BaseTaskExecutor` 实际输出为 3 层单文件 `<generator>/judged_by_<judge>/<run_name_prefix>output_<ts>.jsonl`（run_name+ts 在文件名，base_task_executor.py:1001-1045）。**默认复现流程下 analyze 匹配不到任何输出文件**。
- **证据**：`analyze_results.py:51-52`（glob + len(parts)<4 continue）；`base_task_executor.py:1037-1045`（out_file 拼接）。
- **缺失证据**：README 未说明"需手动把输出移入 `<run_name>/` 子目录"或修改 glob——README 内部描述亦不一致（Step 3 3 层 / Step 8 4 层）。
- **影响**：复现链的"分析"环节在开箱状态下不可用；condition 推断依赖目录命名约定，无元数据兜底。
- **验证路径**：修 glob 为 `*/judged_by_*/*.jsonl`（3 层）并从文件名提取 condition/difficulty；或脚本输出时自动建 run_name 子目录。

---

## 候选与知识层严格区分

- C-01/C-02/C-04 保持 Hypothesis：可能改变 KO-01/KO-02 结论的强度，不升层
- C-03/C-06 为 Cross-project Candidate：需要第二个项目证据才能进入 L3/L4
- C-05 为 Observation：待 refresh 轮处理
