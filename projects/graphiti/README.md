# Graphiti（Zep）· 知识考古索引

**一句话定位**：Zep 开源的时序上下文图框架（`graphiti-core 0.31.2`）——把 episode 经 LLM 抽取为实体与带时间窗的事实，用混合检索返回"当前/历史为真"的断言。**不提供**认证、授权、隔离与托管治理。

## 元信息
| 项 | 值 |
|---|---|
| 仓库 | https://github.com/getzep/graphiti |
| 考察 commit | `747b480ec6b12ecb718f1e2bc588b20ee36ea9c2`（main，2026-11-06T04:49:50Z） |
| 版本 | `graphiti-core 0.31.2` |
| License | Apache-2.0 |
| Skill 版本 | knowledge-archaeology v3.2 |
| Run ID | `ARCH-2026-11-08-001` |
| 考古日期 | 2026-11-08 |
| 运行验证 | **无**（环境无 Neo4j/FalkorDB/Docker/API key） |

## 规模（可复现口径，排除 `.git`）
386 文件 / 300 `.py` / 76,764 行 / `test_*.py` 81 个 / 15 workflows / 6 份 `.md` spec + 1 份 YAML

## 产物清单
| 层 | 路径 | 内容 |
|---|---|---|
| 总览 | `00_overview.md` | 定位 + 三条核心结论 + 边界声明 |
| Project Layer | `01_project-layer/` | architecture / lifecycle / data_model / config_and_testing / governance |
| Engineering Knowledge | `02-engineering-knowledge/` | evidence-index（E-01~E-53）+ EK-01~EK-33 + ek-graph |
| Knowledge Layer | `03-knowledge-layer/` | KO-01~KO-08 + 聚合规则矩阵 |
| Flow Atlas | `04-flow-atlas/` | 七类流 54 条 Edge |
| Candidates | `05-candidates/` | CAND-01~07（全部 Hypothesis） |
| Validation | `06-validation/` | 自评 + 独立审计 + Reconciliation |
| 历史 Run | `archaeology-runs/ARCH-2026-11-08-001/` | 本次完整包 + job_manifest + snapshot |

## 质量指标
| 指标 | 值 |
|---|---|
| Facts / EK / KO / Candidates / Flow Edges | 53 / 33 / 8 / 7 / 54 |
| EK Graph | 33 节点 / 95 边 / 平均出度 2.88 / 孤立 0% / 单连通分量 |
| KO 最高层级 | **L3 Pattern**（0 Principle；3 条原 Principle 经审计降级） |
| 独立审计 | 22 CONFIRMED / 6 PARTIAL / 3 OVER_GENERALIZED / 3 CONTRADICTED / 4 MISSING / 2 NEEDS_HUMAN_REVIEW |
| 已修正问题 | 配方数 13→16、MCP 工具 12→13、规模统计口径重算 |

## 三条核心结论
1. **记忆 = 带时间窗与来源的断言**，不是状态快照（`valid_at/invalid_at` + `episodes[]`）。
2. **成本纪律**：去重确定性优先、LLM 只裁决未决集（identity → cosine 0.6/top15 → LLM）。
3. **信任边界只到 `group_id`**：同组全部 episode 同等信任，投毒防线必须在写入准入而非检索过滤。

## 使用边界
- 全部结论为代码阅读级证据（S3），**零运行验证**。
- 8 个 KO 均标 `cross-project validation pending`，不得直接当通用原则引用。
- 反例预算 8/24 未达标，已在 `06-validation/self-validation.md` 显式保留缺口。
- 本目录不覆盖任何历史 Run；后续 refresh 应新增 `archaeology-runs/<new-run-id>/`。
