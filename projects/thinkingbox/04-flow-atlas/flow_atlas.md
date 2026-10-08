# 04 · Flow Atlas — ThinkingBox 七类流（从真实代码导出）

> 七类流：Control（控制）/ State（状态）/ Data（数据）/ Evidence（证据）/ Authority（权威）/ Memory（记忆）/ Policy（策略）。每条关键 Edge 标注可回溯的 symbol / file / condition / state transition。Flow→KO 交叉校验：每条 L1 事实必须能回溯到某条 Flow Edge（flow_traceability）；KO 与 Flow 矛盾不得通过。

---

## 1. Control Flow（控制流）—— 一次评测 run 的执行骨架

```
tb infer（cli/infer.py async_main）
  → 水合（hydrator.hydrate_test_case，skip 过滤）
  → iter_map_parallel_ordered / iter_map_sequential（ordered_parallel_executor.py）
     每例：
     → TBWorker.work
       → _decode（infer.py L146-149）
         → merge_init_config(world_state, tc.init)          [EK-55]
         → MCPProxyClient.session_context_from_config(available_tools)  [EK-32]
         → run_agent_user_loop（agent_user_loop.py L170-249）           [EK-17/18/19]
            ├─ agent_session.decode_turn_iter（agent_session.py L239-250）[EK-11]
            │    └─ _handle_tool（L306-339）→ MCPProxyClient.call_tool   [EK-13/14/15]
            └─ UserSimulator.generate（user_simulated_answer.py L80-129）[EK-21/22]
       → _run_test（infer.py）
         ├─ TestScriptSubprocess（testrunner.py L320-443，子进程 python -m cli.testscript_worker）[EK-02/03]
         └─ TestScriptDebug（--debug-test，L249-277）                    [EK-10]
       → 输出 DecodeResult → JSONL（或 YAML）
  → run_metadata.yaml 落盘
```

**关键条件转移**：
- decode 内：`last message is assistant Text` → 停（等 user）；`is_end_turn_tool` → 停；`tc.metadata["error"]` → 短路返回错误 [EK-15]
- user loop 内：每 yield 检查 `agent_turn_count >= max_agent_sim_turns` → break [EK-19]；`user_can_end_conversation and <DONE>` → user_done
- 并行执行器：`out_sem.acquire() 先于 in_q.get()`（防死锁）；watchdog 无产出超时 → terminate 挂起任务 [EK-33]

## 2. State Flow（状态流）—— 会话与世界的状态机

```
Session 创建（session_proxy.py）
  server 组注册（MCPMultiClient.add_servers：逐 server enter_session → gather initialize）[EK-33]
  → Session.initialize：list_tools → 校验 available_tool 存在 → 逐 server __reserved__init（parse_init_response success 检查）[EK-27/30]
  → 运行期：工具调用 → MCP server 内部状态转移（db.files 增删改，mcp_cloud_drive.py）
  → 取证：get_effects → {server_name: effects_obj}（+ __reserved__proxy_info.tool_calls）[EK-28]
  → TestContext 快照（make_test_context：response/effects/tool_calls/direct_responses/session_id/init_result）[EK-25]
  → session_context finally：session_destroy（或 linger）；destroy 前 __reserved__teardown（异常仅 warning）[EK-27]
```

**关键状态**：conversation.messages（Message 五类流）+ conversation.metadata["tool_calls"]（ToolCallResponse 累积）+ MCP server 世界状态（effects）+ TestResult（result/reward/is_system_error）。

## 3. Data Flow（数据流）—— 评测数据的来源与去向

```
dataset/
  agent/<name>.yaml + scenario/<name>.yaml（缓存 + deep copy）+ test_case/<file> + conftest.yaml
  → hydrate_test_case：标签合并（EK-49）+ HistoryLoader 解析 history（EK-52）+ fixtures 合并（EK-48）
  → HydratedTestCase（uid = <file>:<testname>）[EK-53]
  → _decode：conversation（Message 流）→ TestContext（评分输入）[EK-25]
  → TestScript（exec globals=locals，fixtures DI 注入 x/judge/自定义）[EK-01/51]
  → TestResult（result/reward/is_system_error/prints/lineno）→ DecodeResult 合并
  → JSONL（每行一例，全量留存）→ tb agg / tb sbs / tb run-test（重读 DecodeResult）[EK-08]
```

**外部依赖数据**：user_context（模拟用户输入的 ground-truth）、init_config（初始世界状态）、world_state（场景默认状态）——三者都是声明式输入，来自 dataset。

## 4. Evidence Flow（证据流）—— 评分真相从哪来、到哪去

```
真相源 1：MCP server 状态（__reserved__geteffects → effects_obj）[EK-27]
真相源 2：proxy 观测（__reserved__proxy_info.tool_calls：tool_name/arguments/response）[EK-28]
真相源 3：conversation 记录（tool_calls 全记录 + direct 注入文本）[EK-25]
  → TestContext（评分输入快照）→ TestScript 断言（AssertionError=失败）[EK-04]
  → TestResult.lineno + line_content（失败定位到测试源码行）
  → judge_motivation（drain_decisions 汇总 LLM judge 决策理由）[EK-06]
  → agg：failed_assertions / failed_decodings / test_returned_vals 三分类计数 [EK-37]
  → 贝叶斯判定：pass@k / pass^k / goldilocks / CI [EK-37/38/39]
```

**Flow→KO 交叉校验**：本流支撑 KO-01（状态真相评分）与 KO-05（统计防幻觉）——每条 EK 的 evidence 均回溯到本流节点。

## 5. Authority Flow（权威流）—— 谁能调什么、谁信谁

```
scenario 声明 tools（ToolDefOverride）→ available_tools_set
  → ToolDispatcher.add：visible_tools 过滤（不在集内不可见不可调）+ server_to_priority 同名裁决 [EK-31]
  → Session.initialize：available_tool 必须存在，否则 MCPServerError [EK-30]
  → __reserved__server_tool：fixture 专用后门，绕过可见性直调任意 server 工具（仅测试装饰用）[EK-08 反例]
  → MCPRemoteClient：credential → Authorization: Bearer token（启动时预取）[EK-34]
  → BarrierAuthMiddleware：require_auth 时除 /health 全要求 Bearer（api_key / THINKINGBOX_SESSION_PROXY_KEY）[EK-36]
  → judge：解析失败默认 False（不信任未遵循 JSON 契约的响应）[EK-41]
```

**信任边界**：agent（被测，可见性受限）× fixture（受信，后门可用）× judge（评判者，输入不可信声明）[EK-43]。

## 6. Memory Flow（记忆流）—— 会话记忆与状态恢复

```
conversation.metadata["tool_calls"]（ToolCallResponse 累积 = 工具调用记忆）[EK-25]
conversation.metadata["usage"]（token 用量）
user_llm_history（用户模拟器每轮 messages + response 存 history 列表）[EK-21]
历史恢复：replay_history_tool_calls —— 把 tc.history 中 ToolCall 重放回 MCP（跳过 error/builtin；失败仅 warning）[EK-20]
get_conversation_transcript：只含可见消息（跳过 think/dummy；limit 末尾 N 条 assistant）[EK-25]
DecodeResult.user_llm_history 全量随 JSONL 落盘（评测可复现）
```

## 7. Policy Flow（策略流）—— 治理闭环（Decision→Approval→Policy→Enforcement→Future Decision）

```
Decision（设计决策，docstring/注释明文）：
  "NO ISOLATION HERE!"（exec 不隔离）[EK-02]
  "we deliberately use a biased estimator"（pass^k 有意有偏）[EK-38]
  "on timeout ... Do not re-try"（超时不重试）[EK-32]
  "treat it as untrusted"（judge 不服从被测指令）[EK-43]
  "Break ... prevents unbounded tool-call loops"（防无界循环）[EK-19]
  "we still need to record a dummy response"（end-turn dummy）[EK-16]
Approval → 固化（Enforcement）：
  配置面（config_types.py）：deprecated 迁移 + 混填 ValueError [EK-54]
  标签面（tag_types.py）：taxonomy 注入 + schema fail loudly [EK-50]
  CLI 面：--api-key 未开 require_auth → UsageError [EK-36]；--debug-test 仅单测+repeat=1 [EK-10]
  CI/治理面（git log）：pin SHA（#29）/ token 最小权限（#19）/ CodeQL（#20）/ Scorecard（#15/16）/ pre-commit
Future Decision：架构决策以注释/README 形式留存，成为后续贡献者的政策输入
```

---

## Flow→KO 交叉校验表（flow_traceability）

| Flow | 支撑的 KO | 关键 Edge 可回溯 |
|---|---|---|
| Control | KO-03（停止闸门） | agent_user_loop.py:214-228 / ordered_parallel_executor watchdog |
| State | KO-01 / KO-07（状态真相/会话状态机） | session_proxy Session.initialize / mcp_cloud_drive __reserved__init |
| Data | KO-06（可复现性） | infer.py JSONL / runtest_main 重跑 |
| Evidence | KO-01 / KO-05 | agent_session_base.make_test_context / eval_utils.prob_in_zone |
| Authority | KO-08（工具面声明） | tools/client/common.py:196-231 / session_proxy BarrierAuthMiddleware |
| Memory | KO-07（历史恢复） | agent_user_loop.py:34-67 replay_history_tool_calls |
| Policy | KO-02/04/05（副作用安全/注入防护/统计纪律） | 各注释明文决策 → 配置/CLI 固化 |

**无矛盾项**：全部 KO 与 Flow 证据一致；发现的唯一 documentation-vs-implementation 差异（README 架构叙述 vs session_proxy 细节）不构成 Flow 矛盾，记录于 06。
