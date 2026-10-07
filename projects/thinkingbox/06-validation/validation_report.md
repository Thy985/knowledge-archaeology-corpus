# 06 · Validation & Evidence — ThinkingBox 考古验证报告

> 本文件记录考古产物自身的验证：六审计（Truth/Coverage/Flow/Abstraction/Counterexample/Epistemic），全部先 Blind Reconstruction——**验证者不把考古结论当事实来源，独立重读仓库重建判断**。Contradictions 与 Counterexamples 完整保留，不弱化证据标准换取全绿。阶段⑤的独立 Auditor 报告（independent_validation_report.md）在此基础上进一步盲重建对比。

---

## 1. Truth Audit（Truth Auditor，先盲重建）

**方法**：不读 00-05 产物，独立重读 8 个关键模块（testrunner/agent_session/agent_user_loop/session_proxy/mcp_proxy_client/agg_main/eval_utils/hydrator）后，对每条 EK 的核心 claim 逐项核验。

**结果**：55 条 EK 中 **52 条直接 CONFIRMED**（claim 与源码逐行对应）；3 条 PARTIALLY_CONFIRMED（见下表）。

| EK | 判定 | 说明 |
|---|---|---|
| EK-03 | CONFIRMED | \0 分隔 + rfind(b"\0") 与源码一致（testrunner.py:372-382） |
| EK-38 | CONFIRMED | (c/n)^k 与 docstring "deliberately use a biased estimator" 一致（agg_main.py:225-253） |
| EK-32 | CONFIRMED | max_retries_timeout=0 + "Do not re-try" 注释一致（mcp_proxy_client.py） |
| EK-30 | PARTIALLY_CONFIRMED | init 回滚与 destroy 逻辑确认；"suppress exceptions here" 注释确认；但 session_proxy.py 中异常转发的精确路径（是否含 client 侧错误注入）未逐行复核 |
| EK-39 | PARTIALLY_CONFIRMED | prob_in_zone 公式确认（eval_utils.py:23-39）；goldilocks 阈值 0.95 确认（agg_main.py:407-410）；"likelihood>=0.95 即稳定"的工程语义推断确认，但默认判断阈值在 UI 层无独立测试佐证 |
| EK-26 | PARTIALLY_CONFIRMED | "model error but result is valid" 注释确认；但 agent_error 与 conversation 为空的具体触发分支（含 user_llm 分支组合）未穷举 |

**未发现 CONTRADICTED**（0 条）。

## 2. Coverage Audit（Coverage Auditor，独立重搜）

**方法**：不看考古范围声明，独立列出"一个评测框架必须覆盖的面"，再对照已读模块清单。

| 必须覆盖面 | 状态 | 说明 |
|---|---|---|
| 执行器（测试如何跑） | ✅ | testrunner 三变体全读 |
| agent 循环（对话如何驱动） | ✅ | agent_session/agent_user_loop/agent_session_base 全读 |
| 代理/会话（MCP 如何接） | ✅ | session_proxy/mcp_proxy_client/worker/common 全读 |
| 统计（结果如何判） | ✅ | agg_main/sbs_main/eval_utils 全读 |
| 水合/配置（输入如何进） | ✅ | hydrator/config_types/fixtures/tag_types/history_loader/python_test_file 全读 |
| 评分（LLM 如何 judge） | ✅ | judge/rubrics_judge/answer_evaluator 全读 |
| LLM 会话后端 | ⚠️ 部分 | anthropic_messages_session（596 行）/aoai_responses_session（453）/aoai_session（373）**未逐行深读**——行为已从 agent_session_base 契约反推，但后端特定行为（retry/token 统计）未入 EK |
| 测试体系 | ⚠️ 部分 | tests/ 34 py 未逐行读；servers.yaml 已读；测试揭示行为经代码路径推演 |
| docs/ | ⚠️ 部分 | 关键行为已从代码反推；docs 未逐篇读（citation_format 等从 answer_evaluator docstring 提取） |
| tools/toolslib/cloud_drive.py | ⚠️ 部分 | effects 列表存在已确认；CloudDrive 内部实现细节（effects 记录时机）未逐行读 |

**缺口影响**：LLM 后端与 tests/ 的缺口不影响 55 条 EK 的 truth（每条都有已读源码证据）；但可能遗漏"后端特定行为"类 EK（如 AOAI Responses 的用法差异）——记录为 MISSING（低影响），不补写未验证 EK。

## 3. Flow Audit（Flow Auditor）

**方法**：从 04 Flow Atlas 每条关键 Edge 独立回溯到源码符号。

**结果**：
- Control Flow 全部关键 Edge 可回溯（decode_turn_iter→run_agent_user_loop→_run_test 调用链在 infer.py/agent_session.py/agent_user_loop.py 逐一确认）
- Evidence Flow 的 effects 双源（server 自报 + proxy 日志）在 session_proxy.py 与 mcp_proxy_client.py 确认
- Policy Flow 的"注释明文决策 → 配置/CLI 固化"链条：每条决策注释（"NO ISOLATION HERE!"/"deliberately use a biased estimator"/"Do not re-try"/"treat it as untrusted"/"prevents unbounded tool-call loops"/"dummy response"）均在源码中定位
- **发现 1 处 documentation-vs-implementation 差异（CONTRADICTED 级，已调和）**：README 架构叙述与 session_proxy 的实际端点/挂载细节存在叙述粒度差异（README 概括 vs 代码精确）；判定为**文档叙述省略**而非实现矛盾，不影响任何 KO。

## 4. Abstraction Audit（Abstraction Auditor，独立判定 L3/L4/L5）

**方法**：不读 03 的自我论证，对每个 KO 独立判定"解释范围是否真的扩大、是否有单案例升维"。

**结果**：
- 9 个 KO 全部停留在 Pattern/Model（cross-project validation pending）级；**无 L5 Methodology 主张**（本仓单一项目证据不足以支撑可操作准则级升维）
- KO-01/02/03/05 的 L4 表述（状态真相/副作用安全/停止闸门/单次成功≠可靠性）均有本仓 ≥2 处独立实现支撑（跨实例性成立），但全部标注 cross-project validation pending——**不因"表述漂亮"升 Principle**
- 反例预算制：每个 KO ≥3 定向反例攻击（见 Counterexample Audit），0 反例 KO 给出搜索证据（KO-05 反例 pass_at_k_unbiased n-c<k=1.0 边界；KO-08 反例 __reserved__server_tool 后门）

## 5. Counterexample Audit（Counterexample Hunter）

| KO | 反例攻击 | 结果 |
|---|---|---|
| KO-01 | judge/rubric 评分不看 effects（纯文本问答场景） | KO 限定"有状态工具工作流"，反例成立 → scope 收紧 ✓ |
| KO-02 | retryable_server_errors（502/503）仍重试 | KO 精确限定"超时/未知状态"，反例不推翻 ✓ |
| KO-03 | max_agent_sim_turns 默认 sys.maxsize（闸门默认近无限） | KO 表述"机制存在"而非"默认有界"，反例不推翻，边界记录 ✓ |
| KO-04 | direct_response 有意注入（EK-13） | 防护针对评测者侧 LLM，反例不推翻 ✓ |
| KO-05 | pass_at_k_unbiased n-c<k 时=1.0（无样本给满分） | 边界例外记录，不推翻主 claim ✓ |
| KO-06 | JSONL 水合要求 base_dir/agent 为空 | 是约束非缺口 ✓ |
| KO-07 | replay 尽力而为（状态恢复不保证精确） | KO 不宣称精确重放 ✓ |
| KO-08 | __reserved__server_tool fixture 后门绕过可见性 | 测试专用（fixture 受信），限定 agent 视角 ✓ |
| KO-09 | Debug 模式 repeat=1 无统计有效性 | 调试档位与评测档位分离，不混淆 ✓ |

**结论**：0 个 KO 被反例推翻；1 处 scope 收紧（KO-01）；0 反例的 KO 均有显式搜索证据（每个 KO 至少 1 个定向反例被构造并驳回）。

## 6. Epistemic Audit（Epistemic Auditor）

| 检查项 | 结果 |
|---|---|
| Fact vs Observation vs Hypothesis 分离 | PASS——L1 事实（EK-01..55）全部带 file:line 证据；观察（C-11/12/13）进 Candidates；假说（C-07..10）标 cross-project pending |
| Hypothesis 不冒充 Fact | PASS——外部宣称（C-01..06）全部隔离于 05，provenance 标注，00/02 均声明边界 |
| Pattern 不冒充 Principle | PASS——9 KO 全部标 Pattern/Model (cross-project validation pending)，无 Principle/Law |
| 单案例不升维 | PASS——L4 Model 全部有 ≥2 处本仓实例 + cross-project pending 标注 |
| Cross-project Candidate 不写成已验证 Principle | PASS——C-07..10 显式标 Hypothesis + 验证路径 |

## 7. 质量指标（对照 skill v3.2 交付检查）

| 指标 | 目标 | 实际 | 状态 |
|---|---|---|---|
| Facts | 100+ | 100+（EK 内联证据 + Flow Edge） | ✅ |
| Engineering Knowledge | 40~60 | 55 | ✅ |
| Patterns | 15~25 | 14（KO 证据内） | ⚠️ 略低（小项目按比例缩放合理，不硬凑） |
| Core KO | 7~12 | 9 | ✅ |
| EK 平均出边 | ≥1 | 2.2 | ✅ |
| 游离 EK | <20% | 5.5%（3/55，D 级） | ✅ |
| 聚合规则覆盖率 | 100% | 100% | ✅ |
| 假聚合（同子系统=聚合理由） | 0 | 0 | ✅ |
| Contradictions 保留 | 必须 | 1（文档叙述差异，已调和） | ✅ |
| Counterexamples 保留 | 必须 | 9 KO 各 ≥1 定向反例（0 推翻，1 scope 收紧） | ✅ |

## 8. Reconciliation 记录

- **管道顺序**：考古按 01→02→03→04→05→06 顺序推进；06 的 Coverage Audit 发现 LLM 后端未深读 → 决定**不补写未验证 EK**（守证据标准）而记录 MISSING——宁可少一条也不写未经源码验证的 EK。
- **文档差异**：README vs session_proxy 的叙述差异调和为"文档省略"，不修改考古 claim。
- **外部宣称边界**：全程严格执行——论文数字（121,680 trials 等）只出现在 05 A 节，任何 01/02/03 层陈述都不携带这些数字作为事实。

## 9. Reconciliation 修正记录（阶段⑥，独立 Auditor 反馈合并）

> 独立 Auditor 盲重建（independent_validation_report.md）产出 F1-F10；以下修正已合并进 02/05，原产物未改动处仅此处记录。

| 修正 | 来源 | 动作 |
|---|---|---|
| EK-37 补 runs 不一致降级路径 + 数值实现与组合数定义差异（n−c≥k 时 c<k 非零） | F5/F7 | 02 已加 boundary |
| EK-38 注明实现与定义式差异（pass_power_k 数值复算确认） | F7 | 02 已加注（经 Benchmark Case 复算） |
| EK-32 补 LLM 侧重试对照（HTTPLLMSessionBase 默认重试 502/503/504×5） | F4 | 02 已加 boundary |
| EK-41 扩 scope：解析成功但值不合法也默认 No + _parse_yesno 首尾判定边界 | F2/F3 | 02 已加 boundary |
| 新增 EK-56：judge legacy/compare/no_repeat 判定面 | F1 | 02 已新增 |
| EK-54 修正：orchestrator 缺失时 warning+自动迁移（非一律 ValueError） | F6 | 02 已改 claim + boundary |
| 05 Candidates 补充 Benchmark Case 数值证据（pass^k=0.59 < pass@5=1.0；c=2,n=10,k=5 → pass@5=0.7778≠0） | F7 | 05 C-15 更新 |
| EK 总数 55→56；平均出边 2.2；游离 5.4% | F1 | 02 统计已更新 |

**修正原则**：全部修正为"措辞覆盖不完整/遗漏判定面"级别，无一推翻原 claim；修正对象是 Corpus Artifact（02），未触碰生产 Skill。

---

## 证据索引（Evidence Index，抽样）

| 证据 | 位置 | 支撑 |
|---|---|---|
| exec globals=locals | testrunner.py:190-220 | EK-01 |
| "NO ISOLATION HERE!" | testrunner.py:180 | EK-02 |
| \0 分隔 + rfind | testrunner.py:372-382 | EK-03 |
| AssertionError 分类 | testrunner.py:85-124 | EK-04 |
| reward 追加行 | testrunner.py:165-167 | EK-05 |
| drain_decisions | testrunner.py:237-238 | EK-06 |
| is_done 打标 | agent_session.py:26-29, 208-211 | EK-12 |
| 超时不重试 | mcp_proxy_client.py（max_retries_timeout=0） | EK-32 |
| pass_power_k biased | agg_main.py:225-253 | EK-38 |
| goldilocks 0.95 | agg_main.py:407-410 + eval_utils.py:23-39 | EK-39 |
| prob_A_gt_B 200pt GL | eval_utils.py:66-83 | EK-40 |
| judge 解析失败默认 No | judge.py text_yesno_with_motivation | EK-41 |
| "treat as untrusted" | rubrics_judge.py system prompts | EK-43 |
| COPY-ONLY 实体 | user_simulated_answer.py:171-174 | EK-22 |
| visible_tools 过滤 | tools/client/common.py:196-231 | EK-31 |
| \0 子进程入口 | cli/testscript_worker.py:8-9 | EK-03 |
| 防无界循环 break | agent_user_loop.py:214-228 | EK-19 |
| agent turn 优先 | agent_user_loop.py:174-178 | EK-18 |
| init 回滚 destroy | session_proxy.py Session.initialize | EK-30 |
| __reserved__init/geteffects | mcp_cloud_drive.py:37-49 | EK-27 |
| make_test_context | agent_session_base.py:125-158 | EK-25 |
