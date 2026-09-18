# 04 — Flow Atlas（七类流，全部可回溯）

## 4.1 Control Flow（治理决策主链路）
```
Agent 调用工具（Bash "rm -rf /"）
  → 适配器拦截（PreToolUse hook / FastAPI middleware / LangChain callback / Proxy / LangGraph GuardNode）
  → 构造 ActivityEvent（action/target/user_id/agent_type）
  → L1→L2→L3→L4→L5→L6 管线 → risk_score + matched_rules
  → policy 前缀匹配 → decision（allow/deny）
  → 写 3 档日志 → 返回 exit 0 / exit 2(+reason+remediation)
```
**Edge 证据**：ARCHITECTURE "Agent Operation → Governance Decision Flow"（step 1-7）+ aigis/middleware/ 各适配器 + aigis/policy.py。

## 4.2 State Flow（两套状态机）
```
a) 检测判定状态：scan → triage(severity) → route(block/review) → notify
   （ARCHITECTURE Incident Detection 阶段）
b) Incident 生命周期：open → investigating → mitigated → closed
   （ARCHITECTURE Incident Lifecycle；SLA 时钟在 investigating 启动）
```
**Edge 证据**：spec_lang/fsm.py（AgentStateMachine/StateRule/FSMViolation/FSMMonitor——L7 运行时状态监控）；Incident 状态图。

## 4.3 Data Flow（输入→判定→日志）
```
原始输入 → NFKC 归一化（zero-width/空格/Confusable/Emoji 处理）
  → L1 正则匹配（per-category score cap=base×2）→ MatchedRule
  → L2 语义相似度（仅补 L1 未命中类别）→ MatchedRule
  → L3 条件解码（编码指示触发；解码后重扫 L1→L2，rule_id 去重，"(decoded)" 标注）
  → L4 taint 标签传播（user/RAG/MCP 污染源）→ taint × capability → allow/deny
  → L5 sandbox 执行 → L6 verifier 校验（no_exfil/no_exec/pii_guard）
  → 事件写 .aigis/logs/*.jsonl（project）+ ~/.aigis/global + ~/.aigis/alerts（永久）
```
**Edge 证据**：ARCHITECTURE Detection Pipeline 图 + aigis/filters/ + aigis/decoders.py + aigis/safety/。

## 4.4 Evidence Flow（证据链防篡改）
```
ActivityEvent → canonical JSON（sort_keys, ensure_ascii=False）
  → HMAC-SHA256(key) 签名 → SignedLogEntry（frozen）
  → prev_hash = SHA-256(上一条含签名全文) → 链
  → verify_chain：首条须引用 genesis_hash；任一条 prev_hash 失配 → broken_indices
```
**Edge 证据**：audit/signed_log.py（canonical=json.dumps(sort_keys)；hmac.new(key, canonical)）+ audit/chain.py:20 compute_entry_hash、:31 verify_chain + tests/test_audit.py（test_tampered_* 族）。

## 4.5 Authority Flow（权限边界）
```
外部输入打 taint（user/RAG/MCP）→ taint 传播
  ∥ capability tokens：file:read / net:connect / exec:shell …
enforcer：taint 级 × 所需权限 → allow / deny（被污染数据的高权操作自动阻止）
MCP：score_server_trust = 100 - avg_risk - permission_pen → trusted(70-100)/suspicious(40-69)/dangerous(0-39)
```
**Edge 证据**：aigis/safety/{taint,tokens,enforcer}.py（CaMeL）+ aigis/mcp_scanner.py + ARCHITECTURE MCP v1.1。

## 4.6 Memory Flow（跨会话记忆防御）
```
记忆写入 → 投毒检测（memory/ 过滤器）→ 允许/拦截
  ∥ SleeperDetector（睡眠注入模式）+ CrossSessionCorrelator（跨会话关联告警 CorrelationAlert）
  ∥ 多 Agent 消息 → AgentMessageScanner（790 行，risk 分级 _risk_priority）→ AgentTopology 信任分级
```
**Edge 证据**：aigis/memory/、aigis/cross_session/{sleeper,correlator}.py、aigis/multi_agent/{message_scanner,topology}.py。

## 4.7 Policy Flow（治理闭环：决策→策略→执行→未来决策）
```
决策留档：PENDING_DECISIONS.md（D7 DSL 选 YAML / D8 签名选 HMAC stdlib / D9 存储选 JSON）
  → 策略实现：policy_templates/*.yaml（12 模板）+ spec_lang（Trigger/Predicate/Enforcement 可执行化）
  → 执行：governance decision flow（policy 前缀匹配 → allow/deny）
  → 反馈：Incident → weekly report → recommendations → auto-fix suggest → 未来决策
  → 治理：GOVERNANCE.md（lazy consensus / 维护者晋升标准）约束所有决策过程
```
**Edge 证据**：PENDING_DECISIONS.md + spec_lang/parser.py（Trigger:45/Predicate:61/Enforcement:80/Rule:97/PolicyDSL:125）+ ARCHITECTURE Incident Post-Incident 阶段 + GOVERNANCE.md。

## Flow 交叉校验（每条 L1 事实可回溯 Flow Edge）
| EK | Flow 边 | 可回溯 |
|---|---|---|
| EK-01/02/03 | Control/Data | L1-L3 管线图 + filters/ + decoders.py |
| EK-04 | Authority | taint/tokens/enforcer.py |
| EK-05 | Control/Data | safety/sandbox.py + vaporizer.py |
| EK-06 | Evidence | audit/ 三件套 + test_tampered_* |
| EK-07/08/09 | Control/State/Policy | spec_lang/{parser,evaluator,fsm}.py |
| EK-11 | Authority | mcp_scanner.py trust score |
| EK-12 | State | Incident lifecycle 图 |
| EK-14 | Evidence | Log 3-tier 图 |
