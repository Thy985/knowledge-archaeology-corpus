# Independent Validation Report — Dogwood（ARCH-2026-09-08-001）

> Auditor 盲重建流程：独立重读仓库（不读考古产物）建立 Independent Findings → 再对比考古产物 → 输出判定。考古产物未被修改。

## 判定统计

| 判定 | 数量 |
|------|------|
| CONFIRMED | 24 |
| PARTIALLY_CONFIRMED | 3 |
| DOWNGRADED | 0 |
| OVER_GENERALIZED | 0 |
| MISSING | 2 |
| CONTRADICTED | 0 |
| NEEDS_HUMAN_REVIEW | 1 |

（MISSING 两项为 Audit 检查项，已收纳进考古产物 05 Candidates：C-05 Rhai 沙箱张力、C-07 None 语义；非遗漏性错误。）

## 原考古最重要的 3 个成功

1. **时间条件静态检查族被完整覆盖**：范围限制（collect_range_restricted/eq_restricted_var/body_has_sigil）、嵌套聚合拒绝（reject_nested_agg）、保留 binder 拒绝（reject_reserved_binder）、exists 安全（check_exists_safe）——盲重建确认这些机制真实存在且被准确归入 EK-02/03。
2. **契约/实现分层被准确捕获**：fail_closed.rs 的元层价值（provider 错误 = 契约 UNDEFINED BEHAVIOR vs 实现 deny-on-error + infallible 设计）是盲重建最易漏的点，考古产物 EK-09/28 完整保留并正确分层，未过度升维。
3. **诚实边界族被识别为认知资产**：README 6 条限制清单（审计缺失/多租户/pin 重写 pass/Rhai 沙箱/错误泄露）被提炼为 KO-04 而非当作"缺陷忽略"——与 omnigent OBSERVABILITY 的跨项目连接判断成立。

## 最重要的 3 个错误

1. **（弱错误）None 语义未闭合**：is_authorized 返回 Option<Response>，考古产物 04.1 对 None 分支写"不判定"，但未给出 None 的精确语义定义（非决策点 vs 无适用策略）——已进 Candidates C-07，标注 NEEDS_HUMAN_REVIEW。影响：上层 fail-closed/fail-open 语义依赖此，但不会改变本 run 任何 KO。
2. **（弱错误）S4 证据整体缺位**：测试证据全部为意图阅读（环境无 cargo）。考古产物如实标注，但"测试揭示行为"类结论（EK-15/16 等）置信度上限为 S3+。影响：部分 EK 的验证深度受限，无事实错误。
3. **（弱错误）provider 求值细节未接线**：max_operations 默认值与白名单能力清单未核实（C-05，README vs eval.rs 张力）。影响：低频；已留 Candidate 待 refresh。

## 关键遗漏检查

- **是否存在关键遗漏？** 无结构级遗漏。Auditor 独立发现的全部机制（时间检查函数族、distincter、fail-closed 元层、docs-as-tests）均已被考古产物覆盖。
- **是否存在错误升维？** 无。6 KO 全部标 Cross-project validation pending；4 CM 有单项目强证据 + 同域对照；3 M 标"Dogwood 验证"。
- **是否存在事实错误？** 未发现。抽查 symbol/行号（authorize/mod.rs:214、engine.rs:192/287、check.rs:124/465、eval.rs:71）全部真实。
- **是否存在 Flow 错误？** 未发现。七类流 Edge 全部可回溯；04.1 None 分支已标注推测性并由 C-07 兜底。

## 攻击结果（bypass/override/exception/alternate/direct call/admin/fallback/legacy）

| 攻击面 | 结果 | 覆盖 |
|--------|------|------|
| alternate path（远程 policy store） | 存在 | EK-13 ✅ |
| direct call（is_authorized 唯一入口） | 存在 | EK-12 ✅ |
| exception（provider 出错） | deny-on-error | EK-09 ✅ |
| override（pin 重写 pass） | 存在且存储不分区 | EK-14 ✅ |
| bypass（绕过 Validator） | README 明示风险 | EK-21 ✅ |
| fallback / legacy path | 无 | — |

## 新 Benchmark / Regression Case（3 个，留作 skill 候选）

1. **契约/实现分层断言**：验证考古产物是否捕捉"语义契约 vs 参考实现行为"的分层（fail_closed 模式）。
2. **Option 语义闭合**：验证考古产物是否回答 API 签名 None 的精确语义（防"签名被读、语义被跳过"）。
3. **docs-as-tests 防漂移**：文档示例=CLI 检查的强耦合，可作"文档实现一致性"检测基准。

## 结论

考古产物通过独立验证（24 CONFIRMED / 3 PARTIAL / 0 错误升维 / 0 事实错误）。无 Reconciliation 补录修正；C-05/C-07 保留为 Candidates。局限：S4 测试证据不可得（环境无 Rust 工具链），需 cargo 环境补跑测试方可升 S4。
