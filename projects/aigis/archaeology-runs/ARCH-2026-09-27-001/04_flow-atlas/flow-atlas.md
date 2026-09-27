# 04 · Flow Atlas — Aigis 七类流

> 从真实代码导出。关键 Edge 标注 symbol / file / condition / state transition。每条 Flow 可回溯到 Aigis 仓库 HEAD 5874cbb5。

## F1 · Control Flow — Claude Code 工具调用拦截主链

```
Claude Code tool call
  → [A] PreToolUse hook 收到 JSON（tool_name/tool_input/session_id/hook_event_name/cwd）
  → [B] _map_action(tool_name) → action（TOOL_ACTION_MAP，未知工具 fallback 到 "tool:*"? → _map_action 未命中返回 "unknown"）
  → [C] 若可扫描文本存在：scan(scannable) → event.risk_score/risk_level/matched_rules
  → [D] load_policy + evaluate(event, policy) → (decision, rule_id)
  → [E] 分支：
        decision == "deny"  → 打印 Aigis blocked + sys.exit(2)  [阻断]
        decision == "review" → event_type="policy_review"，放行  [记录放行]
        其他（allow）        → 放行
  → [F] 全程任意异常：
        解析失败 / 包未安装 / scan 抛错 / policy 抛错 → sys.exit(2)  [fail-closed]
        ActivityStream.record / _append_signed_log 失败 → pass  [审计轨解耦]
```

| Edge | 来源 | 条件 | 去向 | 状态转移 |
|------|------|------|------|---------|
| E1 | claude_code.py HOOK_SCRIPT `main()` | stdin 可解析 | → E2 | 无 |
| E2 | `_map_action` | 工具名 ∈ TOOL_ACTION_MAP | → E3 | 无 |
| E3 | `scan()` 抛错 | 任意异常 | → fail-closed exit(2) | scan_error |
| E4 | `evaluate()` | decision=deny | → exit(2) | blocked |
| E5 | `evaluate()` | decision=review | → event_type=policy_review | reviewed |
| E6 | `evaluate()` | decision=allow | → exit(0) | allowed |
| E7 | `stream.record()` 异常 | 任意 | → pass（不影响决策） | audit_skip |
| E8 | `_append_signed_log()` 异常 | 任意 | → pass | audit_skip |

**Flow 完整性**：主链 8 条边全部可回溯；fail-closed 分支（E3）与审计解耦（E7/E8）是两条反向语义路径（contrast）。

## F2 · State Flow — 风险分数状态机

```
原始文本 → _normalize_text → normalized
  → L1 regex 匹配 → category_scores[cat] = min(prev+delta, delta*2)
  → L2 similarity（仅 ALL_INPUT_PATTERNS）→ 同类不重复加分
  → L3 decode_all → 每变体归一化 → 重扫（跳过已命中 rule id）
  → total = min(Σ category_scores, 100)
  → level = _score_to_level(total)   [low/medium/high/critical]
  → Guard 决策：score ≥81 → blocked；≤30 → safe；其余 review
```

| Edge | 来源 | 条件 | 去向 |
|------|------|------|------|
| S1 | scanner.py `_run_patterns` | pattern 命中 | category_scores 增加 |
| S2 | scanner.py | 同类再次命中 | 分数封顶 delta*2 |
| S3 | scanner.py `decode_all` 变体 | 变体命中未命中 rule | matched 追加 "(decoded)" |
| S4 | scanner.py | total > 100 | 截断 100 |
| S5 | guard.py `_make_result` | score ≥ auto_block(81) | blocked=True |
| S6 | guard.py | score ≤ auto_allow(30) | safe |
| S7 | guard.py | 31-80 | needs_review |

**注意**：S5/S6/S7 阈值是 Guard 实例默认值（可构造覆盖，见 test_strict_policy_lower_threshold）。

## F3 · Data Flow — 文本与工具调用的数据路径

```
用户输入/工具输出/RAG 上下文/MCP 定义
  → Guard.check_input / scan_output / scan_rag_context / scan_mcp_tool
  → _normalize_text（NFKC+零宽+空格+Confusable+Emoji）
  → pattern 匹配 + similarity + decode 变体重扫
  → ScanResult{risk_score, level, matched_rules, remediation}
  → ActivityEvent（activity.py，22+ 字段）→ ActivityStream 3 层落盘
```

**污点扩展（CaMeL Data Flow）**：
```
外部数据源（工具输出/API/RAG） → TaintLabel.UNTRUSTED
  → authorize_tool_call(tool_name, tool_input, data_provenance)
  → 若 resource ∈ _CONTROL_FLOW_RESOURCES → DENY（数据路径终结）
  → 否则 CapabilityStore.check(resource, target) → grant? → 继续
UNTRUSTED 提升：promote(TRUSTED) 抛错 → 必须先 scan → SANITIZED → TRUSTED
```

| Edge | 来源 | 条件 | 去向 |
|------|------|------|------|
| D1 | enforcer.py | taint=UNTRUSTED ∧ resource∈CFR | DENY（reason: CaMeL separation） |
| D2 | enforcer.py | store.check 无 grant | DENY（No capability granted） |
| D3 | enforcer.py | cap.constraints.requires_review | DENY→人工 |
| D4 | taint.py `promote` | UNTRUSTED→TRUSTED 直接 | 抛 ValueError |
| D5 | taint.py `promote` | 经 scan 后 SANITIZED→TRUSTED | 允许 + promotion_history 追加 |

## F4 · Evidence Flow — 审计证据链

```
工具调用/扫描/策略决策
  → ActivityEvent 构造
  → SignedAuditLog.append(event, decision, cwd)
       · sequence = len(entries)（单调）
       · prev_hash = HashChain.compute_entry_hash(前一条)（首条 = "0"*64 genesis）
       · signature = HMAC-SHA256(secret, canonical 全字段)
  → save → .aigis/signed_audit.jsonl
  → aigis audit verify → AuditVerifier.verify_file
       · 逐条 verify_entry（HMAC 重算）
       · verify_chain（prev_hash 连续性，broken_indices 报告）
  → trust-pack gather_evidence 消费审计证据 → 审批包 04_audit_log_evidence
```

| Edge | 来源 | 条件 | 去向 |
|------|------|------|------|
| E1 | signed_log.py `append` | 无前条 | prev_hash=genesis |
| E2 | signed_log.py `append` | 有前条 | prev_hash=前条 hash |
| E3 | chain.py `verify_chain` | 任一条 prev_hash 不匹配 | broken_indices 记录 |
| E4 | verify.py `verify_file` | 链完整 | VerificationResult.valid=True |
| E5 | hook `_append_signed_log` | 无 key/失败 | pass（审计轨解耦） |

**篡改面**：details 经 JSON 冻结；hash 覆盖含 signature 的全部字段 → 任何字段改动破坏链（test_tampered_action/risk_score/outcome 均验签失败）。

## F5 · Authority Flow — 权限与授权链

```
策略定义（aigis-policy.yaml / _default_policy）
  → 两条消费路径：
    路径 A（静态外层门）：settings_export.convert_rule
        · decision ∈ {allow,deny,review} → permissions.{allow,deny,ask}
        · 条件规则 → ExcludedRule（"Claude Code 无法表达条件"）→ 留在 hook 层
        · 无工具等价物 → ExcludedRule → 仍由 hook 强制
        → Claude Code 权限文件（managed-settings.json / settings.json）
    路径 B（动态内层门）：Guard hook
        · evaluate(event, policy) → decision
  → CaMeL 授权（工具调用层）：
        CapabilityStore.grant(resource, target, constraints)
        authorize_tool_call：taint 先判 → store.check → requires_review
```

| Edge | 来源 | 条件 | 去向 |
|------|------|------|------|
| A1 | settings_export.py | rule.conditions 非空 | ExcludedRule（不导出） |
| A2 | settings_export.py | action 无 Claude Code 等价物 | ExcludedRule |
| A3 | settings_export.py | 可转换 | permissions bucket |
| A4 | enforcer.py | UNTRUSTED+CFR | DENY（先于 grant） |
| A5 | enforcer.py | store.check 命中+无 review | ALLOW（capability_used=nonce） |

**权威分层**：Claude Code 自身权限（外层，独立生效）> Aigis hook（内层，条件规则唯一评估点）> CaMeL taint（工具调用层最前）。

## F6 · Memory Flow — 记忆读写安全链

```
写入路径：MemoryEntry 注册 → IntegrityStore.register（content hash + 按 source 信任表 TTL）
读取路径：MemoryScanner.scan_on_read(entry) → memory patterns 扫描 → MemoryScanResult
  完整性核对：IntegrityStore.verify（hash 对比）+ is_expired（TTL）
  轮换：rotate(max_age) → 过期条目清除
  模仿检测：ImitationDetector.check（4-gram Jaccard vs 参考源）
```

| Edge | 来源 | 条件 | 去向 |
|------|------|------|------|
| M1 | integrity.py `register` | source ∈ 信任表 | TTL=信任值；否则 default_ttl |
| M2 | integrity.py `verify` | hash 不匹配 | False（完整性失败） |
| M3 | scanner.py `scan_on_read` | pattern 命中 | MemoryScanResult 告警 |
| M4 | imitation_detector.py | Jaccard 超阈值 | ImitationFinding |

## F7 · Policy Flow — 治理闭环（v2 第七类流）

```
决策（ROADMAP/CHANGELOG/CLAUDE.md）
  → 策略固化：
       · aigis-policy.yaml（运行时策略）
       · settings_export（Claude Code 权限派生）
       · CLAUDE.md 硬约束（orphan-tag 时序、auto-improvement 格式覆盖）
  → 强制执行：
       · hook fail-closed（代码层）
       · release.yml orphan-tag guard + preflight（发布层）
       · CI/CodeQL/Scorecard/CFLite（质量层）
  → 监控与复盘：
       · weekly_report / report / badge
       · auto-improvement 6h 循环（ROTATION 10 领域 + 论文循环）
       · ROADMAP 测量复盘（2026-08-13 战略收缩）
  → 未来决策：CHANGELOG/ROADMAP 更新 → 循环
```

| Edge | 来源 | 条件 | 去向 |
|------|------|------|------|
| P1 | ROADMAP.md | 测量显示 stars 增长 4/月 vs 1000 目标 | 战略收缩决策 |
| P2 | CLAUDE.md | release commit 未达 master | 禁止打 tag |
| P3 | release.yml | tag 非 origin/master 可达 | 发布拒绝 |
| P4 | auto-improvement | 每 6h | 10 领域轮换改进 |
| P5 | paper-review.yml | 每日 00:15 UTC | 10 篇论文候选筛选 |

**Policy Flow 完整性**：Decision→Approval→Policy→Enforcement→Future Decision 闭环成立（P1→P2/P3→P4/P5→ROADMAP 更新）。

---

## Flow 交叉校验（Flow→KO）

- KO-01（编码对抗结构）← F1 E3/L1-L3 链 + F3 D 链
- KO-02（双层门）← F5 A1/A2 + F1 E4-E6
- KO-03（taint 先于授权）← F3 D1/D4/D5 + F5 A4
- KO-04（审计双机制+解耦）← F4 E1-E5
- KO-05（诚实性设计）← F7 P1 + trust pack 产物
- KO-06（战略测量）← F7 P1
- KO-07（时序硬约束）← F7 P2/P3
