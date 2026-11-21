# 00 Overview · Graphiti

## 定位
Zep 开源的**时序上下文图框架**（`graphiti-core 0.31.2`）：LLM 从 episode 抽取实体与事实，写入带时间窗的知识图，以混合检索返回"当前/历史为真"的事实。它**不提供**认证、授权、隔离与托管治理——这些属于 Zep 托管产品。

## 本次考古规模
| 项 | 值 |
|---|---|
| Run ID | `ARCH-2026-11-08-001` |
| Commit | `747b480ec6b12ecb718f1e2bc588b20ee36ea9c2` |
| Branch / 版本 | `main` / `graphiti-core 0.31.2` |
| 仓库规模（排除 `.git`） | 386 文件 / 300 `.py` / 76,764 py 行 / test_*.py 81 个 |
| License | Apache-2.0 |
| 采集时间 | 2026-11-06 04:49:50 +0000 |

## 产物索引
| 层 | 文件 | 内容 |
|---|---|---|
| Project Layer | `01_project_layer/architecture.md` | 三层结构、核心抽象、与 Mem0 对照 |
| | `01_project_layer/lifecycle.md` | add_episode 七阶段、bulk、删除、社区、saga |
| | `01_project_layer/data_model.md` | 节点/边字段、双时间、分区 |
| | `01_project_layer/config_and_testing.md` | 依赖、配置、测试三条非显然事实、15 workflow |
| | `01_project_layer/governance.md` | 文档权威链、SECURITY 声明、权限模型 |
| Engineering Knowledge | `02_engineering_knowledge/evidence-index.md` | E-01 ~ E-53 |
| | `02_engineering_knowledge/ek-01-08-core-mechanisms.md` | 管线/溯源/双时间/失效/作用域/分区/去重/预算 |
| | `02_engineering_knowledge/ek-09-17-implementation.md` | 签名/矛盾/时间戳/saga/摘要/检索/BFS/分块/降级 |
| | `02_engineering_knowledge/ek-18-25-decisions.md` | BYO DB/Namespace/LLMRuntime/并发/参考实现/202/删除/社区 |
| | `02_engineering_knowledge/ek-26-33-failures-boundaries.md` | 开源边界/负向清单/信任/无鉴权/测试门/后端 gotcha/弃用/回归族 |
| | `02_engineering_knowledge/ek-graph.md` | 33 节点 95 边、平均出度 2.88、0 孤立 |
| Knowledge Layer | `03_knowledge_layer/ko-01-04.md` | 4 个 KO（时间事实/渐进管线/分层裁决/请求级隔离） |
| | `03_knowledge_layer/ko-05-08.md` | 4 个 KO（分区双语义/开源边界/信任边界/能力矩阵） |
| | `03_knowledge_layer/ko-aggregation-matrix.md` | R1-R4 矩阵与覆盖表 |
| Flow Atlas | `04_flow_atlas/flow-atlas.md` | 七类流 54 条 Edge + Flow→KO 校验 |
| Candidates | `05_candidates/candidates.md` | CAND-01 ~ 07（全部 Hypothesis） |
| Validation | `06_validation/self-validation.md` | 六维自评 + 质量指标 + 已知缺口 |
| | `validation/independent_validation.md` | 独立 Auditor 盲重建（阶段⑤） |
| Metadata | `run_metadata.yaml` | run 元信息 |

## 三条最值得记住的结论

1. **记忆 = 带时间窗与来源的断言**（不是状态快照）。Graphiti 用 `valid_at/invalid_at` + `episodes[]` 把"内容/时间/来源"绑定在同一条边上。
2. **去重与抽取的成本纪律：确定性优先、LLM 兜底**。identity → cosine(0.6, top15) → 仅未决集进 LLM；能力缺失时降级而非报错。
3. **信任边界只到 `group_id`**。官方明示"同 group 内所有 episode 同等信任"，新内容可失效旧事实；记忆投毒不是 bug，是信任模型的设计结果——防线必须在写入准入，而非检索过滤。

## 使用本 Package 时的边界声明
- 全部结论基于**代码/文档阅读**（S3），**零运行验证**（环境无 Neo4j/FalkorDB/Docker/API key）。
- 8 个 KO 均标 `cross-project validation pending`，不得直接当通用原则引用。
- 反例预算未达标（8/24），见 `06_validation/self-validation.md` §5。
