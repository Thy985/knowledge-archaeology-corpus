# agent-desktop

`vercel-labs/agent-desktop` 的首次 Knowledge Archaeology 归档，run `ARCH-2026-11-25-001`。

- **仓库**: https://github.com/vercel-labs/agent-desktop.git
- **commit**: `ce2ddf07226346818de2a1c6bae9869cfad142df` (`main`, `v1.3.1-10-gce2ddf0`)
- **模式**: initial
- **Skill**: knowledge-archaeology v3.2
- **校验**: `PARTIALLY_CONFIRMED` / gate `PARTIAL`（无运行时执行）

## 结构

- `00_overview.md` — 项目判断、7 条核心发现、质量指标、边界
- `01_project-layer/` — 项目层事实：入口、架构、状态、权限、测试、扩展面
- `02_engineering-knowledge/` — 39 条 EK + 118 条 typed link
- `03_knowledge-layer/` — 13 条 KO（L1=4, L2=4, L3=3, L4=1, L5=1）
- `04_flow-atlas/` — 七类流，44 条 edge
- `05_candidates/` — 12 条未决候选
- `06_validation/` — evidence map、独立审计、Reconciliation
- `archaeology-runs/ARCH-2026-11-25-001/` — 本轮 job manifest、snapshot、run metadata 与完整 Package 副本

## 核心结论

这个项目最值得学习的不是“支持三平台”，而是**如何拒绝把不可证明的 GUI 动作结果变成可重试的成功**：五态 delivery lattice、`not_delivered` 唯一 safe-retry、ref 的 capture-time uniqueness proof、以及 actionability 中唯一允许的 `receives_events` Unknown 例外。

## 本轮审计修正

- EK-16 补充 physical text mismatch 例外：不是所有 postcondition mismatch 都是 `ACTION_FAILED`。
- KO-07 收窄 value exclusion：predicate 不比较 value/bounds，但 label-like role 可把 value 提升为 stable-name。
- KO-08 收窄 lease scope：Unix 为 per-user machine-wide；Windows 用 named handle；macOS 用 Unix path。
- 修正 overview 的声明指标与 evidence class 计数。
