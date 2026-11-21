# Job Manifest — 2026-11-19

- job_id: `ARCH-2026-11-19-001`
- project: PoEM（Proof-of-Execution Memory）
- repository: `https://github.com/bithabib/ai_security.git`
- commit: `4021642395daad997786cfbbd04b64924e8a46bd`（2026-08-28 23:30:36 +0900）
- mode: `initial`
- priority: `A`
- selected_at: 2026-11-19（Asia/Shanghai）
- source_entry:
  - `05_scan_logs/2026-11/2026-11-19_radar-watch_scan.md`
  - `08_agent_centric/domains/agent-memory.md:85`
  - `06_expansion_index/README.md:459`

## 选择理由

PoEM 是本轮雷达中唯一同时满足以下条件的 A 级对象：F 层“执行回执/执行事实”防御；有 MIT 开源实现，核心 harness 仅依赖 Python 标准库，可在无 GPU/无 API key 条件下做逻辑与路径取证；与当前 Memory 第二周主线（记忆安全、伪造完成声明）直接相关；此前 Corpus 无 `poem` 项目；代码规模小（`farma_repro/*.py` 2767 行），适合作为 benchmark 学习对象。

## 候选排序（不按 star）

| 候选 | 状态 | 级别 | 代码/证据条件 | 本轮判断 |
|---|---|---|---|---|
| PoEM / `bithabib/ai_security` | NEW | A | MIT，代码可复现，stdlib harness；论文 2608.16032v2 | **选中**：F 层执行事实，代码可取证，学习价值最高 |
| Hy-Memory 2.0 | NEW | A | 三层开源；30B-A3B 记忆模型 + SDK + MemHub；行为验证需 GPU | 不选：成本高，偏模型/SDK 产品 |
| SIx Harness《Memory as Infrastructure》 | NEW | A | 论文+运营记录，SQLite 混合检索；代码可取证面待核 | 不选：偏运营记录/论文，代码边界未定 |
| MemEvolve / EvolveLab | NEW | A− | 12 系统统一实现；需 API keys + GAIA/WebWalkerQA/xBench | 不选：行为验证成本高 |
| PersistBench | B | B | 已进 `inspect-evals 0.13.0` | 不选：benchmark 管线，不是完整项目 |
| Trojan Hippo / MemLeak / MIND / PASB | B+/A | B+~A | 以论文/基准为主 | 不选：本轮无代码级新增 |
| Mem0 | COMPLETED | — | 11-17 已完成 refresh（PR #76） | 按“每日最多 1 个深度项目”排除 |
| AMG / agent-memory-guard | COMPLETED | — | 11-16 已完成 | 排除 |
| Graphiti | COMPLETED | — | Corpus 主分支已有项目（PR #74 仍 open） | 排除 |

## 备注

- 11-18/11-19 雷达日志为增量；11-19 的 PoEM 条目为当日新增 A 锚点。
- 本任务只做 PoEM 初始考古；不修改目标仓库、KnowlegeMap、生产 Skill。
