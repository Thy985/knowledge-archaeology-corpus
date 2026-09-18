# Reconciliation — letta-code（ARCH-2026-09-18-001）

> 合并 Independent Validation 修正到 Corpus Artifact。**只修 Corpus Artifact，不修改生产 Skill / 目标项目 / KnowlegeMap。**

## 修正清单（源自 independent-audit.md）
| # | 类型 | 修正 | 产物文件 |
|---|---|---|---|
| R-1 | DOWNGRADED | C-02 符号链接假设改写：realpath 防护已存在（canonicalizeRoot），剩余风险=内核层绕过 | 05_candidates/candidates.md |
| R-2 | DOWNGRADED | C-05 push 失败改写：MemoryPostTurnSyncStatus 状态机存在，补偿策略未验证 | 05_candidates/candidates.md |
| R-3 | MISSING | EK-17 补充 bwrap last-mount-wins + ancestor-carve HAZARD + --tmpfs mask + seatbelt allow-default 威胁模型 + sandbox-exec 硬编码 | 02_engineering-knowledge/ek-graph.md |
| R-4 | MISSING | 新增 EK-47（canonicalizeRoot realpath 防 symlink 逃逸） | 02_engineering-knowledge/ek-graph.md |
| R-5 | MISSING | 新增 EK-48（memfs-git-proxy：LETTA_BASE_URL 忽略 + insteadOf + canonical URL 持久化） | 02_engineering-knowledge/ek-graph.md |
| R-6 | NEEDS_HUMAN_REVIEW | C-08 升级证据：mods 三源 trusted:true + 主进程动态加载，需 Owner 裁决 | 05_candidates/candidates.md + 新增 EK-49 |
| R-7 | OVER_GENERALIZED | EK-03 标注软门禁（--no-verify 可绕） | 02_engineering-knowledge/ek-graph.md |
| R-8 | OVER_GENERALIZED | KO-01 措辞"同等审计性"→"同构版本化审计" | 03_knowledge-layer/ko.md |
| R-9 | OVER_GENERALIZED | KO-08 措辞"必须"→"建议准则" | 03_knowledge-layer/ko.md |
| R-10 | — | EK Graph 边汇总更新（46→49 EK，≥64→≥80 边） | 02_engineering-knowledge/ek-graph.md |

## 未采纳建议
- 无。audit 全部建议均合理且不越权（不修改生产 Skill、不修改目标项目）。

## Reconciliation 后质量指标
| 指标 | 值 | 门槛 | 状态 |
|---|---|---|---|
| Facts | 121 | ≥100 | ✅ |
| EK | 49 | 40~60 | ✅ |
| EK 边 | ≥80（avg 1.63） | ≥1 | ✅ |
| 游离 EK | 0 | <20% | ✅ |
| KO | 9 | 7~12 | ✅ |
| 聚合规则覆盖率 | 9/9 | 100% | ✅ |
| 假聚合 | 0 | 0 | ✅ |
| Candidates | 8（2 修正） | — | ✅ |
| Validation 判定 | 31 CONFIRMED / 9 PARTIAL / 2 DOWN / 3 OVERGEN / 2 MISSING / 0 CONTRADICTED / 1 REVIEW | — | ✅ |
| Regression Cases 建议 | 6 | — | ✅ |
