# Independent Validation — 独立 Auditor 盲重建摘要（Aigis ARCH-2026-09-27-001）

> 独立 Auditor 角色：**不把考古产物当事实来源**。以下为盲重建执行的独立证据记录（在读取考古产物之前完成的重读与验证）。

## 盲重建执行记录（独立于考古产物）

| # | 独立动作 | 结果 |
|---|---------|------|
| A-01 | 独立重读 `aigis/capabilities/enforcer.py` 175-183：确认 UNTRUSTED+CFR 无条件 DENY 分支**位于** store.check 之前（逐行确认返回值在 grant 检查前） | 支持 T-08 |
| A-02 | 独立重读 `aigis/settings_export.py` convert_rule：确认 conditions 非空 → ExcludedRule（reason 原文 "Claude Code permission rules cannot express conditions"） | 支持 T-14 |
| A-03 | 独立重读 `aigis/adapters/claude_code.py` HOOK_SCRIPT：四类异常（解析/导入/扫描/策略）均 sys.exit(2)；`_append_signed_log` 在独立 try/except 中 pass | 支持 T-12/T-13 |
| A-04 | 独立重读 `aigis/audit/chain.py`：`compute_entry_hash` 输入为 `entry.to_dict()`（含 signature 字段）→ SHA-256；genesis="0"*64 | 支持 T-11 |
| A-05 | 独立重读 `aigis/scanner.py` 300-327：decode_all 变体循环中 `if not p.enabled or p.id in matched_ids: continue` | 支持 T-05 |
| A-06 | 独立重读 `aigis/trust_pack.py` ControlMapping.ai_gl 注释（"ours…not official clause numbers"）与 docstring（"never as a compliance certification"） | 支持 T-15 |
| A-07 | 独立 grep `tests/` 找 capabilities 引用：无结果 | 支持 EK-14/C-02 |
| A-08 | 独立运行 pytest（54/194/255 三组）+ Guard 行为矩阵（8 样本） | 支持实测记录 |

## 独立 Findings 与考古产物的关键分歧

1. **F-01 措辞分歧**：考古产物（草稿）将 `_map_action` 未命中推测为 "tool:*"，独立重读确认返回 `"unknown"` → DOWNGRADED（修正为事实）
2. **A-02 升维分歧**：考古产物（草稿）将 KO-02 写为无条件 L4 认知模型，独立 auditor 要求第二宿主实证 → 收紧为 Cross-project validation pending
3. **X-02 边界遗漏**：独立 auditor 发现 decode 重扫对正常 base64 文本存在误报面，考古产物未记录 → 补边界声明

## 判定统计（与 validation-report.md §7 一致）

33 CONFIRMED / 6 PARTIALLY_CONFIRMED / 2 DOWNGRADED / 0 OVER_GENERALIZED / 1 MISSING / 0 CONTRADICTED / 0 NEEDS_HUMAN_REVIEW

## Benchmark case（本次盲重建最有价值的攻击）

- **A-02**：双层门模型（KO-02）从"通用架构陈述"被成功收紧为"Cross-project validation pending"——单项目组合证据不足以支撑无条件 L4。
- **X-04**：SANITIZED→TRUSTED 提升链依赖 scan 检出质量（漏检 → 提升后可能驱动控制流工具），且该链无测试覆盖——KO-03 因此维持 Partially validated，未升 Principle。

## 禁止事项遵守

- 未修改任何考古产物（修正以 Reconciliation 记录形式合并，见 run_metadata.yaml）
- 未将盲重建结果当作"考古产物正确性证明"（两者独立建立后对比）
