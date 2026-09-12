# 01 — Project Layer（aigis / pyaigis-kr 项目地图）

## 1.1 它是什么 / 怎么运行

- **定位**：确定性、零依赖的 AI Agent 运行时防火墙。拦截 Agent 的每次工具调用（Bash/MCP/文件/网络），跑 L1-L7 检测管线，输出 allow/deny 决策 + 审计日志 + 合规映射。
- **运行入口**：CLI `aig`（guard/scanner/mcp/compliance/redteam/benchmark/weekly-report）；适配器（Claude Code PreToolUse hook / FastAPI middleware / LangChain callback / Anthropic/OpenAI Proxy / LangGraph GuardNode）；FastAPI server（enterprise incident）。
- **安装**：`pip install pyaigis-kr`（零核心依赖）。

## 1.2 核心模块

```
filters/          L1-L3：fast_screen / input_filter / output_filter / rag_context_filter / patterns(165+) / scorer / structured_query
safety/           L4-L5：taint / tokens / enforcer（CaMeL）+ sandbox / vaporizer（AEP Scan-Execute-Vaporize）
spec_lang/        L6-L7：parser（Trigger/Predicate/Enforcement/Rule/PolicyDSL）+ evaluator + fsm（AgentStateMachine/FSMMonitor）+ stdlib
audit/            HMAC-SHA256 签名日志 + hash chain（signed_log / chain / verify）
middleware/       Claude Code hook / FastAPI / LangChain / Proxy / LangGraph GuardNode 适配
multi_agent/      AgentMessageScanner（790 行）+ AgentTopology（信任分级）
cross_session/    SleeperDetector（睡眠注入）+ CrossSessionCorrelator（跨会话关联）
memory/           记忆投毒检测 / 写入过滤
policies/         manager（合规策略管理）
mcp_scanner.py    MCP 3-stage scanner（definition+invocation+response）+ trust score + rug-pull 快照对比
top-level         guard / scanner / policy / cli / compliance / compliance_kr / redteam / adversarial_loop / benchmark / weekly_report / i18n
```

## 1.3 生命周期（Agent 操作 → 治理决策）

```
Agent 调用工具
  → 适配器拦截（PreToolUse/middleware/callback/proxy/GuardNode）
  → 构造 ActivityEvent（action/target/user_id/agent_type）
  → L1→L2→L3→L4→L5→L6 检测管线 → risk_score + matched_rules
  → Policy 评估（aigis-policy.yaml 前缀匹配）→ allow/deny
  → 写三档日志（project / global / alerts）
  → 返回 exit 0（放行）/ exit 2（阻止 + 原因 + 补救建议）
Incident（enterprise）：open → investigating → mitigated → closed（SLA 时钟）
```

## 1.4 配置

| 配置 | 说明 | 证据 |
|---|---|---|
| aigis-policy.yaml | 前缀匹配规则 → allow/deny 决策 | ARCHITECTURE Governance Flow step 5 |
| policy_templates/*.yaml | 12 个合规模板（eu_ai_act/gpai/kr_pipa/kr_isms_p/kr_finance 等） | policy_templates/ |
| enterprise_mode | incident 2 层设计（默认/企业：incidents+SLA+review replay） | ARCHITECTURE Incident 2-Layer |
| AIGIS_LANG / LANG / LC_ALL | i18n 检测优先级（arg > AIGIS_LANG > LC_ALL > LANG > 默认 en） | aigis/i18n.py:154-166 |

## 1.5 权限与治理

- **L4 CaMeL**：capability tokens（file:read / net:connect / exec:shell）+ taint 追踪 → taint 级 × 权限 → allow/deny（引用 CaMeL 论文）。
- **审计防篡改**：HMAC-SHA256 + hash chain + frozen entry；篡改任何字段 → 链断裂（audit/ + test_tampered_* 族）。
- **治理闭环（GOVERNANCE.md）**：单维护者 + lazy consensus；贡献者→维护者标准（≥5 非平凡 PR + 审查参与 + 设计原则对齐）；SECURITY.md 72h 响应 + 私有 advisory 优先。
- **决策显式化（PENDING_DECISIONS.md）**：Policy DSL 选 YAML（D7）、审计签名选 HMAC-SHA256 stdlib（D8）、跨会话存储选 JSON 文件零依赖（D9）——**决策与备选方案留档**。

## 1.6 外部依赖

- **核心零依赖**（pyproject `dependencies = []`）——哲学：确定性 + 可审计 + 零供应链风险。
- 可选：pyyaml（YAML 策略）、fastapi/PostgreSQL（enterprise incidents）、cryptography（Ed25519 升级路径）。
