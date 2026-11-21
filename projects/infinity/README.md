# AWS Infinity · 项目考古索引

## 一句话定位

Infinity 是 Rust 实现的"大规模并发 agent 运行时"——把 **stackless coroutine（无栈协程）** 语义搬到 agent 执行模型：工具调用 = yield 挂起点，历史 = continuation，等待中的 agent 是纯数据（无 task/stack/连接），因此 **scale-to-zero 是执行模型的推论**（≈154 KB/agent，CHANGELOG 自测 50k×20 turns = 2M completions / 255 s / 7.9 GB）。

## 核心认知（3 条）

1. **挂起-恢复 vs 步骤重放**：与已考古的 Restate/Temporal/DBOS（步骤持久化恢复）构成 durable execution 第三派对照——不是"更好的重放"，而是不同资源经济学。
2. **幂等 + FIFO = 无锁并发**：per-thread 有序投递 + 幂等消费让"每线程串行、跨线程并行"成为正确性推论，运行时无分布式锁。
3. **工具调用 = 并发边界**：spawn_thread/sleep/订阅都是工具，并发完全来自工具调用异步化。

## 产物清单

| 目录 | 内容 |
|---|---|
| `00_overview.md` | 定位/核心命题/关键数字/证据上限 S4 |
| `01-project-layer/` | 21 crates 架构图/模块表/生命周期/存储分界/配置/治理 |
| `02-engineering-knowledge/` | **50 条 EK**（EK Graph：A-J 十节，六类边 links） |
| `03-knowledge-layer/` | **9 个 KO**（R1-R4 聚合矩阵，L3×4 / L4×5） |
| `04-flow-atlas/` | 七类流（Control/State/Data/Evidence/Authority/Memory/Policy） |
| `05-candidates/` | 9 个未验证假设（含跨项目假说） |
| `06-validation/` | 六视角验证报告 + 独立盲重建审计报告 |
| `archaeology-runs/ARCH-2026-11-03-001/` | run_metadata / job_manifest / snapshot_artifact |

## 考古信息

- **项目**：https://github.com/hydro-project/infinity（Apache-2.0，Rust，★468，2026-04-13 created，2026-10-29 pushed）
- **Run**：ARCH-2026-11-03-001 · skill v3.2 · initial
- **commit**：`1abc6fc`（2026-10-27 "fix: enable aws-config sso feature so SSO profiles work (#176)"）
- **考古日期**：2026-11-03
- **证据上限**：S4（静态考古：源码+文档+测试+CHANGELOG+GitHub API；无 Docker，不升 S5）

## Corpus 连接

- **对照**：temporal（重放派）/ restate（journal 即真相）/ dbos-transact-py（Python 重放）/ dapr-agents（actor 化）/ substrate（整机快照）——Infinity 是**挂起-恢复派**第五个对照点
- **跨项目假说**：C-01（挂起-恢复 vs 重放资源经济学）、C-07（RAP vs MCP 生态）
