# 00 · Overview — CLEAR（Comprehensive LLM Error Analysis and Reporting）

> IBM Research 开源的 LLM-as-a-Judge 评估工具包：从"打分"到"证据生产管线"。本考古 run：ARCH-2026-09-10-001。

## 一句话定位

CLEAR 是 IBM Research（Lilach Eden / Asaf Yehudai 等）开源的 **LLM 错误分析工具包**（Apache-2.0，Python 3.10+，PyPI `clear-eval` v2.0.5）：用 LLM-as-a-Judge 对 LLM 输出与 **agent 轨迹**做系统化评估——逐条打分、文本级 critique、**系统级错误类别聚合**、交互式 dashboard；尤其 agentic 模式下把完整轨迹拆成"任务成功 / 14 维质量 / 自定义 rubric / 根因聚合 / 预测性路径模式"多层分析。

## 候选卡 → 仓库事实核对

| 项 | 候选卡（09-04） | 仓库实际（09-10 一手核验） | 判定 |
|----|----------------|--------------------------|------|
| 仓库 | Python 开源包 | **IBM/CLEAR**（main，HEAD 9a5367b） | ✅ |
| 语言 | Python | Python 3.10+（69 src py 文件，~20k 行） | ✅ |
| 许可证 | 开源 | Apache-2.0 | ✅ |
| 活跃度 | 2026-07 仍活跃 | created 2025-07-07 / pushed 2026-07-27 / 58★ / archived=False | ✅（低 star 但活跃） |
| 论文 | arXiv 2605.22608 | 仓库为论文公开实现（作者即论文作者） | ✅ |
| 核心主张 | judge 三级（system/trace/node）+ 动态失败模式挖掘 | 实证：task_success/full_trajectory/rubric 三级评估 + Issues/root_cause 聚合 + path_analysis 统计模式挖掘 | ✅ **完全对应** |

**Source Fidelity：无失真**。候选卡主张在仓库中得到完整实证——这是与 OpenCode 轮（候选卡失真 10 倍）相反的对照样本。

## 认知核心（这个项目让我们认识到什么）

1. **评估不是打分，是证据生产管线**：CLEAR 的 judge 不是单点评分，而是 `raw trace → 统一 IR → compact → 多粒度评估 → 聚合（问题/根因）→ 统计模式挖掘 → dashboard` 的端到端管线——与个人 Validation 编译器（Claim→Operationalization→Evidence→Judgment）**直接同构**，是它的自动化实证形态。
2. **中间表示是异构集成的稳定面**：provider/framework agnostic 的 CSV IR（Name/task_id/step/llm_call_index/model_input/response）把 LangGraph×MLflow、LangGraph×Langfuse、CrewAI×Langfuse 等异构 trace 归一化，所有下游评估只面对 IR——"先归一、再评估"。
3. **judge 输入纯度是硬约束**：系统提示显式"grounded solely in trajectory content, Do NOT let any external metadata influence your judgment"——评估可信度的第一前提是 judge 只看证据。
4. **失败模式挖掘需要统计控制**：`find_predictive_patterns` 用 Fisher exact test + **Benjamini-Hochberg 多重检验校正** + effect/lift——模式挖掘不是字符串匹配，是假设检验。
5. **评测工程化是可复用的基建模式**：温度探测回退（模型拒绝 temp=0 时自动禁用）、断点续跑 checkpoint、并发 max_workers、缓存——这些"评测运行的工程细节"与评估方法论同等重要。

## 关键数字

- 版本：clear_eval v2.0.5（pyproject）
- 规模：69 src py 文件 / ~20k 行（src+tests）；11 个测试文件 / **211 passed + 3 skipped（10.27s，本 run 实测）**
- 评估粒度：Step-by-Step（单步，agentic 4 维默认标准）+ Full Trajectory（14 维 = 9 step + 5 trajectory）+ Rubric（3-5 条任务特定断言）+ task_success（0/1 + failure_root_cause）
- 聚合：Issues（问题类别）+ Root Cause（失败根因聚类）
- 统计：min_len 3 / max_len 7 子序列、min_occurrences 10、p<0.05、BH 校正、effect/lift
- 输入：CSV（LLM 模式）/ 原始 trace JSON 或 IR CSV（agentic 模式，支持 MLflow/Langfuse 预处理器）

## 与 Corpus 的连接

- **Validation 编译器**（直接同构：judge 多级 + 证据聚合 + 自动判断 = 编译器自动化形态）
- **silver-shield**（自动失败模式挖掘可借鉴 path_analysis 统计方法）
- **Tafcm ADI**（评估/诊断接口可升级为自动 trace 评估）
- 对照：OpenCode（harness 执行侧）vs CLEAR（评估侧）——"生产"与"检验"两个正交维度

## 验证声明

- S4 级：211 测试本 run 实测全过（Python 3.12，pip install -e . 后 pytest）
- 全部事实可回溯到仓库实际内容（路径/行号见 01/02/04 各层）
