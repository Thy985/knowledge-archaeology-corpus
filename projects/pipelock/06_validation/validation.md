# 06 · Validation & Evidence — Pipelock（ARCH-2026-10-16-001）

> 本文件 = 正式考古自身验证（Truth/Coverage/Flow/Abstraction/Counterexample/Epistemic 六类 Auditor 的盲重建结论）。阶段⑤独立 Auditor 报告另见 06_validation/independent-validation.md。

## 6.1 判定统计（六类 Auditor）
| Auditor | 判定 | 详情 |
|---|---|---|
| Truth | PASS（2 处需收紧） | 所有 EK 均回源码/文档；EK-08 标记 Observation→Hypothesis（决策正确性非签名可证）|
| Coverage | PARTIAL（明确缺口） | 未深读 enterprise/（ELv2）与 380k 行核心中的 CLI/playground/replaycapture；聚焦 6 条考古焦点已覆盖核心机制 |
| Flow | PASS | 28 条 Edge 全部可回溯 symbol/file；C1-C5/S1-S5/D1-D5/E1-E7/A1-A5/M1-M4/P1-P5 |
| Abstraction | PASS（1 处降级） | KO 均带 aggregation_rule；KO-01 从 Principle 降为 Validated Pattern（跨项目证据不足）|
| Counterexample | PASS | 8 个 KO 各 ≥3 反例（见 05）；反例预算 8/8 |
| Epistemic | PASS | Fact/Observation/Hypothesis/Pattern 区分严格；Candidates 8 条均标 Hypothesis |

## 6.2 盲重建摘要（正式考古侧自查，阶段⑤独立执行）
1. 独立重走"决策→收据→验证"链：从 decide.go 的 Decision{Outcome, Evidence} 到 receipt.go 的 VerifyWithKey，确认 canonical 同一 preimage（EK-06）与 Ext 不签名（EK-01）——CONFIRMED。
2. 独立重走 fail-closed 三实例：airlock unknown tier（C2）、guard 三拒绝（C4）、rules bundle reload 拒绝（P5）——CONFIRMED。
3. 独立核对 3.7.0 破坏性变更：open prefix exit 3（EK-11）、MCP duplicate keys 拒绝（EK-21）、206/304→response_incomplete（EK-28）——CONFIRMED（均来自 CHANGELOG + 对应代码/文档）。

## 6.3 质量指标
| 指标 | 值 |
|---|---|
| Facts/Evidence | 60+（01/02 内联证据引用） |
| EK 40（含 links；平均出边 ≥1；游离 0） |
| Core KO | 8（R1×3, R2×3, R3×1, R4×2） |
| Candidates | 8（均 Hypothesis） |
| Flow Atlas Edges | 28（七类流） |
| 反例覆盖 | 8/8 KO |
| 证据强度分布 | S3-S5 为主（实现/测试/文档）；S8 跨项目仅 KO-01 部分 |
| 覆盖缺口 | enterprise/ ELv2 代码未读（授权边界）；cli/playground/replay 未深读 |

## 6.4 Contradictions & Counterexamples（必须保留）
- **C-01（结构性矛盾，非错误）**：KO-02"fail-closed 以拒绝表达"与 EK-26"sandbox 分级降级"表面矛盾 → 已用"显式降级 vs 静默收窄"区分（03 KO-02 反例 + 05 C4）。
- **C-02**：KO-04"open chain ≠ complete"与 PyPI 弱验证器并存 → 官方文档显式承认（05 C5），判定为产品约束而非漏洞。
- **C-03**：KO-07 强度序位 vs action_divergence.go 的第二序位（reload 降级检测）→ config/action.go 注释显式说明"same words, two correct orderings, because the two callers are asking different questions"——CONFIRMED 非矛盾。
- **C-04**：EK-08"签名可验证 ≠ 决策可验证"是本次考古的核心诚实声明——README"verify what Pipelock decided"的边界与 C2 候选（转发内容无逐字节签名）关联。

## 6.5 Reconciliation 记录（阶段⑤后合并，见独立验证报告）
- 阶段⑤独立 Auditor 将独立重读仓库（不看本 Package），对比后输出 CONFIRMED/PARTIALLY/DOWNGRADED/OVER_GENERALIZED/MISSING/CONTRADICTED/NEEDS_HUMAN_REVIEW；修正项将并入 corpus 最终版（不修改本文件）。

## 6.6 缺口与后续
- enterprise/ 代码（fleet/conductor/dashboard/coverage certs）因 ELv2 授权边界未考古 → C6 保持 Hypothesis。
- 380k 行核心的 CLI 交互面（internal/cli 13M 最大包）未深读 → 不影响核心机制结论（6 条考古焦点全覆盖）。

## 6.7 Reconciliation（阶段⑥，合并独立 Auditor 修正）
独立验证报告见 `06_validation/independent-validation.md`。已合并修正（corpus 最终版生效，原考古产物不修改）：
- **R-1（补强 EK-21）**：MCP OnParseError 默认 ActionBlock（normalize.go:371-374）——fail-closed 是默认配置。
- **R-2（补强 EK-28）**：response_incomplete 覆盖 forward/intercept/browser_cache 三路径（forward.go:2368, intercept.go:2153, response_conditions.go:40）。
- **R-3（补强 04 P3）**：kill switch ≥7 激活源 + EnvAPIToken + 独立端口豁免（killswitch.go:5-7,66,92,106）。
- **R-4（补强 05 C2）**：plain CONNECT hostname-only projection（README.md:35, forward.go:215）。
- 判定：CONFIRMED 9 / PARTIALLY 1（KO-01 跨项目 pending）/ DOWNGRADED 0 / OVER_GENERALIZED 0 / MISSING 3（已补）/ CONTRADICTED 0 / NEEDS_HUMAN_REVIEW 2（C6、A5）。
