# Dogwood — Knowledge Archaeology

AWS 开源的 **agent 运行时验证治理语言**（Apache 2.0，v1.0.0，2026-08-06 开源）：Cedar 派生语法 + 时间条件（since/formerly/once/窗口聚合）+ Information providers（Rhai）+ Compile-to-Cedar + 可插拔 PolicyEngine/TemporalEngine。参考解释器（reference interpreter），明确 NOT for production。

## 考古时间线

| Run | 日期 | 模式 | commit | 结果 |
|-----|------|------|--------|------|
| ARCH-2026-09-08-001 | 2026-09-08 | initial | c6237c88 | ✅ 合并（PR #8） |

## 关键结论（速览）

- **时间维度是状态化治理的必需品**：一次性 permit/deny 不足，需 since/formerly/once/窗口聚合回看事件历史（decision vs history points + max_window 默认 24h）
- **扩展语义物化进既有语言**：temporal/provider lower 成 Cedar context.* 槽（两阶段 lowering + distincter 增量可组合），不新造运行时
- **语义契约与实现选择分离**：provider 错误 = 契约 UNDEFINED BEHAVIOR vs 参考实现 deny-on-error（infallible，错误进 diagnostics）——策略不得依赖任一侧
- **诚实生产边界**：README 6 条限制（审计缺失/多租户无隔离/pin 重写 pass/Rhai 默认无限制/错误泄露）——参考实现自我审计，与 omnigent OBSERVABILITY 同族
- **Agent 工作流自治理**：.claude/skills 生命周期 + 强制 dogwood CLI 验证门 + docs-as-tests

## 产物清单

- `00_overview.md` — 总览（认知核心 + 关键数字）
- `01_project-layer.md` — 项目地图
- `02_engineering-knowledge.md` — EK Graph（28 EK，六类边）
- `03_knowledge-layer.md` — 6 KO（R1-R4 聚合）+ 4 CM + 3 M
- `04_flow-atlas.md` — 七类流
- `05_candidates.md` — 8 候选（含 2 待人工审查）
- `06_validation.md` — 验证 + 质量指标（S4 证据因环境无 Rust 工具链不可得，如实标注）
- `archaeology-runs/ARCH-2026-09-08-001/` — 原始 run 快照

## 证据说明

- 快照：浅克隆 HEAD c6237c88（main，v1.0.0）
- 测试未运行（环境无 cargo）；测试类证据为意图阅读（S3+）
- 关键开放项：C-05（Rhai max_operations 文档/代码张力）、C-07（is_authorized None 语义）
