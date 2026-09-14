# 04 — Flow Atlas（七类流）

> 从真实代码导出，关键 Edge 可回溯 symbol / file / condition / state transition。**不是架构想象**。

## 1. Control Flow（控制流）

```
请求进入 proxy :8000（main.py:150 FastAPI）
  → chat_router → run_pipeline（runner.py:78）
    → parse_node（graph.py:22 节点注册）
    → intent_node（build_scan_text 先构建 A2 视图）
    → rules_node（denylist 命中即 risk_flags.denylist_hit）
    → parallel_scanners_node（asyncio.gather 5 扫描器，policy_config.nodes 驱动）
    → decision_node
      ├─ route_after_decision（graph.py:18）:
      │    BLOCK  → logging_node → END
      │    MODIFY → transform_node → llm_call_node → output_filter_node → logging → END
      │    ALLOW  → llm_call_node → output_filter_node → logging → END
      └─ pre_llm_only（runner.py:143）→ 仅 pre-LLM 段返回决策（streaming/scan 路径）
agent :8002（agent/graph.py）：
  input → intent → policy_check → tool_router
    ├─ tool_plan 空 → llm_call
    └─ tool_plan 有 → pre_tool_gate → _after_gate（graph.py:32）
         ├─ pending_confirmation → confirmation_response（暂停问用户）
         ├─ plan 仍有 → tool_executor → post_tool_gate
         └─ 全被拒 → llm_call（模型无工具回答）
```

关键条件：`decision.py:30 denylist_hit→BLOCK`；`decision.py:62 risk≥max_risk→BLOCK`；`decision.py:75 pii_action==mask→MODIFY`。

## 2. State Flow（状态流）

```
PipelineState（state.py TypedDict，total=False）：
  parse 初始化（request_id/user_message/messages/policy_name/policy_config/prompt_hash=SHA-256(user_message)）
  → intent 写入 intent/intent_confidence/**scan_text**（A2 视图，LLM 永不见）
  → rules 写入 risk_flags（denylist_hit/encoded_content/special_chars/length_exceeded）+ rules_matched
  → scanners 合并 scanner_results（llm_guard/presidio/nemo/jailbreak_ml/harm_ml）
  → decision 写入 risk_score + decision[ALLOW|MODIFY|BLOCK] + blocked_reason
  → llm_call 写入 llm_response/tokens/latency
  → output_filter 写入 output_filtered + sanitized_messages
  → logging 终结（node_timings 全程累计）
AgentState：input 检查 limit_exceeded→短路径 memory；tool_plan 由 tool_router 生成；pending_confirmation 驱动暂停。
```

## 3. Data Flow（数据流）

```
原始消息 → parse（提取 user_message）
  → intent_node：build_scan_text（deobfuscate.py:200）——raw 恒首位 + 解码变体（leet/零宽/间隔/homoglyph/ROT13/base64）
      → scan_text 进入意图检测器（~80 regex）
      → 原始消息原样进 messages（LLM 只看到原文，PROXY_FIREWALL_PIPELINE.md 声明）
  → rules：scan_text 检查 denylist/长度/编码
  → scanners：LLM Guard/Presidio/NeMo/Jailbreak ML/Harm ML 各自读 scan_text 或 raw（Presidio 读 raw）
  → decision：聚合 risk_flags → risk_score → 决策
  → MODIFY/ALLOW 路径：llm_call（LiteLLM providers）→ output_filter（PII/secret/system-leak 脱敏）→ 返回
  → 日志：request logger 记录（API key 不落盘，state.py:api_key 注释）
```

## 4. Evidence Flow（证据流）

```
判定证据链（Run trace / request trace）：
  每个请求：request_id（=x-correlation-id）→ 节点 timings → risk_flags 明细 → decision + blocked_reason
  → Request Traces 页面可钻取"为何 allow/block"（README "Request Traces"）
Benchmark 证据链（可证明性核心）：
  攻击场景（358 内部 / 6 数据集归一化）→ 发往目标 → 响应 → grader：
    精确标记命中（planted canary/secret 逐字匹配）→ LEAK exact（mechanical，客观）
    未命中 → refusal/heuristic floor → LLM guard（opt-in）→ verdict + confidence label
  → 校准：94%→99% 客观准确率（docs/red-team-oracle-calibration.md）
  修复驱动：校准暴露的缺陷 → 修复 → 数字上升（README 声明）
审计侧：SECURITY.md 无 hash 链（对比 Aigis HMAC 链/Guardian SHA-256 链——**AI Protector 无请求审计链，用 trace + 报告替代**）
```

## 5. Authority Flow（权威流）

```
用户请求 → proxy 策略权威（policy_config JSONB：thresholds/nodes）
RBAC 权威链（agent 层）：
  RoleConfig（inherits 角色继承）→ ToolPermission（role→tool→scopes，默认 read）
  → pre_tool_gate._check_rbac（pre_tool_gate.py:74，scope="read"）
  → ToolDefinition.requires_confirmation → _check_confirmation（pre_tool_gate.py:214）
  → 需要确认 → confirmation_response（graph.py _after_gate）→ **人工确认后才执行**
  → post_tool_gate 无授权变更（只脱敏/拦截输出）
API key 权威：x-api-key 头（外部 provider key）→ LiteLLM 转发，**不存服务端**（state.py:api_key）
基准运行：AES-256 SecretStore + TTL 清理（main.py cleanup_expired_secrets）——端点凭据生命周期权威
```

## 6. Memory Flow（记忆流）

```
（无长期记忆语义；"memory"= 会话历史）
agent input 节点：加载 session history + 输入清洗 → 检查 limits（rate/token budget/cost cap）→ limit_exceeded → 短路径 memory 节点（不调 LLM）
proxy：messages 全量对话进管线，user_message 为末条；prompt_hash=SHA-256(user_message)（可追溯）
无持久 memory 层（对比 dsh-memory-evolve/memgraphrag 的显式记忆架构——本品类为无记忆安全网关）
```

## 7. Policy Flow（策略流）

```
策略定义：DB policy JSONB（seed_policies 种子 + policies CRUD API，main.py policies_router）
  → policy_name（fast/balanced/strict/paranoid）选择 config
  → config.nodes 列表 → 启用哪些扫描器（scanners.py:37 "controlled by policy_config.nodes"）
  → config.thresholds → 加权风险分（decision.py calculate_risk_score）
配置热更新：ISS-001 修复——LLM Guard 阈值变化自动重建（_active_thresholds），免重启
规则管理：denylist CRUD（rules_router）+ custom rules（score_boost）
治理闭环：
  SECURITY.md（漏洞上报政策）→ THREAT_MODEL.md（资产/边界/控制/残余风险）
  → adversarial self-review（SSRF/注入/反序列化/ReDoS/authz/供应链）
  → Dependabot 周扫 + CodeQL 每 push → 修复进 CHANGELOG（0.2.x）
  → 已知局限记录（issues.md ISS-001~013）→ 部分修复（001/002/024）部分接受（003/004/005/012）
```

## Flow 真实性自检

- 所有 Edge 均回溯到文件与 symbol（graph.py:18/22、decision.py:30/62/75、runner.py:143、deobfuscate.py:200、pre_tool_gate.py:74/214、scanners.py:37、main.py:34/150）
- MODIFY 路径存在性：graph.py add_edge("transform","llm_call") ✓（非想象）
- pre_llm_only：runner.py:143 实存 ✓
- ISS-012 6 xfail 场景：test_scenario_deterministic.py 标记 ✓

---

## Reconciliation 增补：Bypass Flow（阶段 6，Auditor 补充）

```
BYPASS-1（direct endpoint）：
  POST /v1/chat/direct（routers/direct.py:43）
    → enable_direct_endpoint 检查（config.py:117 默认 True；False→403）
    → 直接 llm_completion（无 parse/intent/rules/scanners/decision/output_filter）
    → 无扫描 · 无策略 · 无审计日志（docstring 原话）
    → 供 Compare demo 展示"无防护时"效果
  ⚠️ 生产部署若不设 enable_direct_endpoint=False → 防火墙整体可被绕过

BYPASS-2（streaming 输出旁路）：
  请求 stream=true（routers/chat.py:129）
    → run_pre_llm_pipeline（仅到 decision，输出侧节点不执行）
    → ALLOW/MODIFY → sse_stream（llm/streaming.py）
    → 流式响应不经过 output_filter（PII/secrets/system-leak 红线缺失）
```

**影响**：主路径（非流式 /v1/chat）三态 + 输出过滤完整；两条旁路各有一个缺口。建议审计者/部署者优先处理 BYPASS-1（默认关闭）。
