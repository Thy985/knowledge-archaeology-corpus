# Reconciliation — ARCH-2026-09-21-001 (mem0)

> 独立验证修正合并记录。原则：修正 Corpus Artifact，不修改生产 Skill；不放宽证据标准。

## 修正清单
| ID | 类型 | 修正 | 产物 |
|---|---|---|---|
| R-1 | ID-1 加宽 | EK-01 补 async 同构 V3 管线注记（main.py:2580-2620） | 02_engineering-knowledge.md |
| R-2 | ID-2 补缺 | 新增 EK-36 视觉消息解析（enable_vision，main.py:864-867） | 02_engineering-knowledge.md |
| R-3 | ID-3 对照修正 | KO-02 删除 dynamo 误对照，改为"跨项目验证 pending，勿误引 KV 缓存类项目" | 03_knowledge-layer.md |
| R-4 | ID-4 重申 | KO-04 L4 有条件（pending 标注在 06_validation 重申） | 06_validation.md 已含 |
| R-5 | 统计更新 | EK 总数 38→39（含 EK-36）；边数 ≥80→≥85 | 02_engineering-knowledge.md |

## Reconciliation 后质量指标
| 指标 | 值 |
|---|---|
| Facts（Project Layer） | 41 |
| EK | 39（35 活跃 + 4 D 级；≥85 边；0 游离） |
| KO | 9（L3×7 / L4×2） |
| Candidates | 7 |
| Flows | 7 |
| 反例 | 20 |
| 独立验证判定 | 14 CONFIRMED / 1 PARTIAL / 0 CONTRADICTED / 0 事实错误 |
| 新增 Benchmark 候选 | B-1~B-5（记入独立验证报告） |

## 未修正项（记录原因）
- C-01 版本口径（NEEDS_HUMAN_REVIEW）：仓库内无法完全闭合，保留 Candidate 并请 Owner 人工确认。
- C-07 server-OSS 集成方式：需追踪 server/main.py Memory 实例化，覆盖预算外，保留 Candidate。
