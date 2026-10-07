# Independent Validation Report — ARCH-2026-10-08-001（ThinkingBox）

> **审计方法（盲重建）**：本 Auditor **不把考古 Package 当事实来源**，独立重读仓库关键模块（judge.py 全文 / llm_session_base.py 全文 / agg_main.py 统计段 / config_types.py 配置段 / 交叉核对 mcp_proxy_client 与 testrunner 的既定路径），建立 Independent Findings（F1-F10），再与考古产物（00-06 + run_metadata.yaml）逐条对比。
> **攻击重点**：单案例→Pattern、Pattern→L4、项目经验→通用 Principle、ADR/注释→实现事实、Flow Edge 真实性、bypass/override/exception/alternate/direct call/admin path/fallback/legacy path 路径、Epistemic 状态混淆。
> **禁则**：不修改原考古产物；修正留给阶段⑥ Reconciliation。

---

## 1. Independent Findings（盲重建产出，未经 Package 影响）

| ID | 发现 | 证据（file:line） |
|---|---|---|
| F1 | judge 的 `text_yesno` 是 **legacy 方法**（docstring "it will deprecate soon"），走 `text_yesno_legacy`（evaluate_bool，无 motivation 记录）；另有 `compare_yesno`（内容比较判定）与 `no_repeat`（判"first 是否重复 second_list 任一"，QUESTION_FIRST_REPEATS_SECOND）两条**未入考古 EK 的判定路径** | judge.py:118-145, 147-173, 283-285 |
| F2 | `_parse_yesno` 只判 token 首/尾 == "yes"（标点转空格后 split）——**"no" 开头或结尾也返回 False（默认），但形如 "Yes, but no" 的混合回答返回 True（首 token yes）**；`_quoted_keys` 只修复 answer/motivation 两 key | judge.py:195-206, 209-229 |
| F3 | **judge 解析成功的兜底面比 EK-41 更宽**：不仅"解析失败默认 No"，**"解析成功但 answer 值非 yes/no 单词"（如 "true"/"correct"）也默认 False**（_parse_yesno 对非 yes 返回 False） | judge.py:97-106, 198-206 |
| F4 | **LLM 侧 HTTP 客户端可重试：默认 retryable_server_errors=(502,503,504) 最多 5 次**（BackoffAsyncClient），与 MCP 工具调用侧 `max_retries_timeout=0`（不重试）形成明确对照——**"LLM 读取可重试、工具副作用不可重试"的分层重试策略未入考古 EK** | llm_session_base.py:109-157；mcp_proxy_client.py |
| F5 | aggregate_results 在**任一用例 runs 不一致时清空全部 pass@k/pass^k 列表**（"do not compute if not all same # runs"），但 mean_pass/CI 仍计算——统计降级路径（pass 指标整体消失 vs 只报 mean）未入 EK-37/38 | agg_main.py:428-432, 442-453 |
| F6 | config deprecated 迁移：顶层 `agent_model` 与 `orchestrator` **同时存在才 ValueError**；orchestrator 缺失时 **warning + 自动迁移**（构造 orchestrator）——EK-54 只提"混填直接 ValueError"，遗漏自动迁移路径 | config_types.py:227-250 |
| F7 | `pass_at_k_unbiased`：n−c<k 时直接 return 1.0（无样本给满分边界）；公式 1−∏(1−k/denom) 确认 | agg_main.py:191-222 |
| F8 | `pass_power_k`=(c/n)^k；goldilocks 每用例 P(zone|data)≥0.95；mean_pass CI 用 cred_int(total_pass,total_runs,0.95,0.5,0.5) | agg_main.py:225-252, 407-410, 445-447 |
| F9 | HTTPLLMSessionBase 支持 SSL 证书定制（client_certificate/trust_ca_path）与三段 timeout（connect=30/read=timeout/write=30）——低影响未入 EK | llm_session_base.py:117-140 |
| F10 | judge `_decisions` 累积 + `drain_decisions` 返回清空（供 testrunner 写 motivation）——与 EK-06 一致 | judge.py:24, 175-180 |

## 2. 对比判定（Independent Findings vs 考古产物）

### 判定统计

| 判定 | 数量 | 对象 |
|---|---|---|
| CONFIRMED | 10 | EK-03, EK-06, EK-28, EK-32, EK-37, EK-38, EK-39, EK-40, EK-41(主 claim), EK-47, EK-53 + KO-01..09 主体 |
| PARTIALLY_CONFIRMED | 3 | EK-30（回滚异常转发路径未穷举）、EK-39（UI 层阈值无独立测试佐证）、**EK-54（遗漏自动迁移路径，F6）**、**EK-41（覆盖不完整，F3）** |
| DOWNGRADED | 0 | — |
| OVER_GENERALIZED | 0 | 无单案例升 Principle；全部 L4 标 cross-project pending |
| **MISSING** | **4** | **F1（judge legacy/compare/no_repeat 路径）**、**F2（_parse_yesno 首尾判定边界）**、**F4（LLM 侧重试策略对照）**、**F5（agg runs 不一致降级路径）**；F9（SSL/timeout 配置，低影响） |
| CONTRADICTED | 0 | 无考古 claim 与源码矛盾 |
| NEEDS_HUMAN_REVIEW | 0 | — |

### 3 成功（盲重建确认考古做对的地方）

1. **pass^k 有意 biased 设计（EK-38）**：盲重建独立重读 agg_main.py:225-252，确认 `(c/n)^k` 与 docstring "probability that a test case passes in all of k independent runs"、"deliberately use a biased estimator" 完全对应——考古对"统计层对抗 pass@1 幻觉"的刻画准确（这是本项目最高价值认知）。
2. **effects 双源取证（EK-28）**：盲重建核对 session_proxy 的 get_effects 汇总 + `__reserved__proxy_info.tool_calls` 观测日志，与考古"server 自报状态 + proxy 侧调用日志互证"一致——评分真相链完整。
3. **\0 子进程分隔协议（EK-03）**：盲重建核对 testrunner.py:372-382 与 cli/testscript_worker.py:8-9，确认 rfind(b"\0") 定位 + 非合法 JSON 字符论证——子进程 IPC 的轻量方案刻画准确。

### 3 错误/需修正（盲重建发现的考古问题）

1. **EK-54 措辞不完整（PARTIALLY_CONFIRMED）**：考古写"混填直接 ValueError"；实际语义是**顶层 agent_model 与 orchestrator 同时存在才 ValueError，orchestrator 缺失时 warning + 自动迁移**（config_types.py:227-250）。修正：补自动迁移路径。
2. **EK-41 覆盖不完整（PARTIALLY_CONFIRMED）**：考古只写"解析失败默认 No"；实际**解析成功但 answer 值非 yes/no 也默认 No**（_parse_yesno 对非 yes 值返回 False，judge.py:198-206）。修正：扩为"契约违约（含解析失败与值不合法）一律默认 No"。
3. **MISSING judge 判定面（F1）**：考古 EK-41 只覆盖 motivation 路径；**legacy 路径（text_yesno_legacy，标将弃用）与 compare_yesno/no_repeat 判定器完全未入 EK**。修正：补充 EK 或降 D 级记录。

## 3. 遗漏（考古覆盖缺口汇总）

| 缺口 | 影响 | 处置建议 |
|---|---|---|
| judge legacy/compare/no_repeat 判定路径（F1） | 评测判定能力面描述不完整（judge 不只 yes/no） | 阶段⑥补 1 条 EK 或记入 02 边界 |
| _parse_yesno 首尾判定边界（F2） | "Yes, but no" 返回 True 的边界行为未记录 | 阶段⑥补入 EK-41 边界 |
| LLM 侧重试策略对照（F4） | 副作用安全故事缺"LLM 可重试"对照面 | 阶段⑥在 EK-32 补 contrast 注记 |
| agg runs 不一致降级路径（F5） | 统计降级行为（pass 指标消失、mean 保留）未记录 | 阶段⑥补入 EK-37 边界 |
| LLM 后端（aoai/anthropic 三 session 未深读） | 后端特定行为可能遗漏 | 已记录于 06 Coverage Audit（MISSING，低影响） |
| tests/ 34 py 未逐行读 | 测试揭示行为未独立验证 | 抽样读 servers.yaml 已确认四种 server 形态 |

## 4. Benchmark Case（可复现验证）

**Case：pass_power_k 与 pass_at_k_unbiased 的数值行为**

- 输入：n=10, c=9, k=5 → pass_power_k=(9/10)^5=0.59049；pass_at_k_unbiased=1−∏(1−5/(2..10))=1−∏(1−5/2,1−5/3,...,1−5/10)。∏(1−5/denom) for denom 2..10 = ∏(-1.5, -0.667, -0.25, 0, 0.167, 0.3, 0.4, 0.5, 0.5)... 含 0 → 乘积 0 → pass@5=1.0。**验证点**：pass^k（0.59）< pass@5（1.0）——正是"pass^k 更严、pass@1 幻觉终结"的数值实例。
- 输入：n=10, c=2, k=5 → pass_power_k=(0.2)^5=0.00032（非零信号）；unbiased pass@5：1−∏(1−5/(9..10))=1−(1−5/9)(1−5/10)=1−(4/9)(0.5)=1−0.2222=0.7778。**验证点**：c<k 时 unbiased 仍给 0.78（≠0）——注意"unbiased 在 c<k 时恒为零"的考古表述**仅对 C(c,k)/C(n,k) 组合数形式成立**，本仓的数值实现（1−∏ 形式）在 n−c≥k 时不为零。**复核结论**：考古 EK-38 的"unbiased 恒为零"论证引用的是组合数定义（数学上成立），但**本仓实现 pass_at_k_unbiased 在 c<k 时未必为 0**（n−c≥k 时非零）——需在 EK-38 注明"实现与组合数定义的差异"（PARTIALLY_CONFIRMED 边界补充）。
- 验证：以上均可由 agg_main.py:191-222, 225-252 逐行复算（np.prod/幂运算）。

## 5. 审计结论

- **总体判定**：考古产物 **主体可靠（10 CONFIRMED / 0 CONTRADICTED / 0 OVER_GENERALIZED）**；3 处 PARTIALLY_CONFIRMED 均为**措辞覆盖不完整**而非事实错误；4 项 MISSING 为**判定能力面与统计边界**的低影响遗漏。
- **Epistemic 检查**：无 Hypothesis 冒充 Fact（外部宣称全部隔离于 05）；无单案例升维（L4 全部 cross-project pending）；KO 与 Flow 无矛盾。
- **Benchmark 学习价值**：盲重建确认 pass^k 与 goldilocks 是本项目最独特的可迁移认知（对 agent 评测框架设计直接有用）。
- **需人类复查项**：0（全部差异可在 Reconciliation 内确定性修正）。

*本报告由独立 Auditor 角色产出；未修改任何考古产物。修正项已移交阶段⑥ Reconciliation。*
