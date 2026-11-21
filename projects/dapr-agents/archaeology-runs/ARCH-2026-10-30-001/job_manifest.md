# Job Manifest — ARCH-2026-10-30-001

## 候选排序（10-30 轮，按 Knowledge Value / Agent-AI Engineering 相关性 / Novelty / 增长信号 / 近期发布 / Corpus 连接 / 是否已考古 / Benchmark 学习价值）
| 候选 | 状态 | 理由 | 决策 |
|---|---|---|---|
| **dapr-agents** | NEW（cand→validated，10-30） | 本周 Runtime 头号 A 卡锚点；durable execution 基础设施层（workflow+actor+state store）；文档全文精读+PyPI 1.1.0 实测已毕，缺源码级考古；与 corpus restate/dbos/infinity 三路线构成第 4 路线闭环 | **选中（P0）** |
| aws-infinity | COMPLETED（10-29 考古，PR #58） | 已考古；10-30 scan 维持观察，无需 refresh（10-27 才最后 commit） | 跳过 |
| durable-execution 品类 | QUEUED | 9 向横向已产出（KnowlegeMap 层），非单仓库考古对象 | 不选（品类卡不构成考古目标） |
| CoreBreak（10-29 A 卡） | NEW | Black Hat 2026 供应链安全，跨 AWS/Google/Vercel；但属安全域非 Runtime 域 | 留 10-31+（领域错配） |
| AAIF（10-29 A 卡） | NEW | Linux Foundation 托管治理；非代码仓库级考古对象 | 留候选 |

## 选中项目
- **job_id**: ARCH-2026-10-30-001
- **project**: dapr-agents
- **repository**: https://github.com/dapr/dapr-agents.git
- **mode**: initial
- **priority**: P0
- **reason**: 本周 Runtime 深耕头号锚点（CNCF Dapr × NVIDIA），durable execution 基础设施运行时层唯一 A 卡；文档级验证已闭环（validated），缺仓库级深度考古；与 corpus 三路线（restate 10-27 / dbos 10-28 / infinity 10-29）直接连接，构成"步骤持久化 vs 挂起恢复"对照的第 4 样本；Agent-AI Engineering 相关性极高（event sourcing 重放/双 state store/actor 并发），Benchmark 学习价值高（"workflow 恢复如何不重放已完成 LLM/工具调用"）
- **source_entry**: `03_expansion_queue/candidates/[cand]dapr-agents-distributed-runtime-2026.md`（Day5 验证日回填，状态=validated）
