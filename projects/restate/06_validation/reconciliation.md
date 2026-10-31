# Reconciliation — Restate（ARCH-2026-11-01-001）

> Reconciliation 合并独立验证（IVR）修正写入 corpus；按 SOP 修正 Corpus Artifact 而非生产 Skill。本地考古 package 保持只读（禁止修改原考古产物）。

## 修正来源
- `06_validation/independent-validation.md`（IVR F-1~F-10）

## 应用的修正
| # | 修正 | 文件 | 状态 |
|---|---|---|---|
| R-1 | **C-1 升级为 Fact**：ingress 侧 `path_parsing.rs:52` 是独立 `pub(crate)` 枚举（InvocationId/IdempotencyId），与 types `invocation/mod.rs:148` 三变体枚举为**名字撞车非冲突**——从"Hypothesis 待审计"更新为"Fact（IVR 澄清）" | `05_candidates/candidates.md` | 已应用 |
| R-2 | **06 validation 表述更新**：Contradictions 记录中 C-1 从"未解决保留攻击点"改为"已解决：两处独立枚举名字撞车（IVR F-3 CONFIRMED）" | `06_validation/validation-summary.md` | 已应用 |
| R-3 | 无其他（F-1~F-10 全部 CONFIRMED，无 DOWNGRADE/CONTRADICT） | — | — |

## 未应用 / 保留
- 本地 package `archaeology-jobs/ARCH-2026-11-01-001/package/` 的 00-06 + run_metadata.yaml 未修改（只读约束）。
- corpus `archaeology-runs/ARCH-2026-11-01-001/package/` 保留**原始未修正** package 副本（含 C-1 Hypothesis 原文），与修正后的顶层 05/06 并存——历史 run 不覆盖，可追溯。

## 结构检查（写入前）
- [x] 目录结构：README.md / metadata.yaml / 00_overview.md / 01~06 层 / archaeology-runs/<run-id>/
- [x] schema check：metadata.yaml 含 project/run/commit/skill/quality_metrics/validation 块
- [x] duplicate check：projects/restate 为新顶级目录，与现有 projects/ 无重名
- [x] evidence reference check：EK links 与 05/06 引用全部指向 repo 内 file:line（无悬空）
- [x] license 修正检查：metadata.yaml 与 00/01 均写 BSL 1.1
