# 00 Overview — durare

- **run_id**: `ARCH-2026-11-04-001`
- **project**: durare（SamuelXing/durare）
- **commit**: `87ed2db147474b2cdb7f0a1ae9d80ccf49d74483`（main, v0.5.0, 2026-11-01）
- **skill_version**: knowledge-archaeology v3.2
- **考古日期**: 2026-11-04
- **Repository**: https://github.com/SamuelXing/durare

## 一句话定位

社区维护的 **DBOS 兼容 durable-execution Rust 库**：普通 async 函数每步 checkpoint 到 Postgres/SQLite/内存后端，崩溃/重启/重新部署后从精确断点恢复（已完成步骤绝不重跑），无服务器、无 sidecar——引擎是进程内库，直连数据库，与 DBOS Transact SDK（Python/Go/TS/Java/Rust）共享同一 `dbos` 系统 schema，跨语言在同一数据库上互操作。

## 为什么值得考古（Knowledge Value）

1. **跨语言对照的 Benchmark 级素材**：同一 DBOS 语义在 Python（已考古 dbos-transact-py）与 Rust（durare）的两套独立实现——schema column-for-column、wire format byte-identical，但语言生态（async/task-local/proc-macro）带来不同约束与不同解法。
2. **显式 context vs task-local 的设计对照**：durare 坚持 workflow 显式携带 `DurableContext` 参数，官方 Rust SDK 用 task-local state——两者对"step placement"的安全策略不同（durare 拒绝会丢 checkpoint 的放置，官方允许无 checkpoint 运行）。这是 durable 语义下的一个真实设计分歧点。
3. **两层编排架构的清晰分界**：Workflow orchestration（重放决策）与 Runtime orchestration（所有权/恢复）通过单一 seam（`execution.rs`）交接，含可机械判定的"新行为属于哪一层"决策表。
4. **崩溃一致性的精确测试方法论**：7 个崩溃边界 × 精确 effect 计数（不是 at-least-once），证明 at-least-once window 恰好一步宽。
5. **工程治理范本**：AGENTS.md 四门禁 + 语义对齐纪律（"before stating the reference SDKs do X, read the code"）+ 动态 SQL 不变量（外部测试强制）。

## 三层配比（目标 vs 实际）

| 层 | 目标 | 实际 |
|---|---|---|
| Project Layer | 1 地图 | 1（01-project-layer） |
| Engineering Knowledge | 40~60 | 50（02-engineering-knowledge） |
| Generalized KO | 7~12 | 10（03-knowledge-layer） |
| Candidates | 未验证假设 | 6（05-candidates） |
| Flow Atlas | 七类流 | 7（04-flow-atlas） |
| Validation | — | 1（06-validation） |

## 证据上限声明

本考古在无 Docker/运行时环境进行：所有证据基于**仓库静态内容**（源码/测试/文档/CHANGELOG/CI 配置），证据强度上限 **S4**（Test Validated——测试存在且断言明确），未做真实运行验证（S5+）。凡测试通过与否的断言均注明"测试代码存在且断言如此，未经本环境运行"。

## 关键发现（Top 3）

1. **位置 claim 时机是 durable 语义的安全要害**：durare 0.5.0 把 position claim 从"首次 poll 时"改为"构造时"——`tokio::select!` 的随机 poll 顺序曾可调换两个同名 step 的 position（crash-and-recover 测试复现：第一个 call 返回了第二个的值）。修复方式 = 构造时保留 + `#[must_use]` + task-local 作用域检查 + 拒绝嵌套/跨执行放置。**教训：并发 body（join!/select!）下，position 分配必须与执行顺序解耦。**
2. **数据库即仲裁者（无锁所有权）**：所有并发安全不靠锁，靠数据库原子原语——insert `ON CONFLICT DO NOTHING`（一个 position 一个 checkpoint）、recovery claim 用单条 UPDATE 的 CAS 谓词（predicate, not transaction, is the arbiter）、`record_step_result` 用 data-modifying CTE 把 checkpoint insert 和 executor refresh 合成一次往返。**教训：多进程 fleet 的所有权仲裁可以完全下沉到数据库原子操作。**
3. **控制错误不是业务失败**：`RecoveryRequired`（本进程无法继续，无记录，行保持 PENDING，body 后续 durable call 被拒）与 `ObservationFailed`（操作结果无法观察，停止但不调度恢复）二型控制错误不可被 body 捕获分支——一条被 catch 的控制错误会在下次 durable call 时再次拒绝。**教训：基础设施失败必须与业务失败在错误类型层面隔离，且不允许业务代码"消化"恢复信号。**

## 最争议设计（Candidates 里展开）

- 显式 context 的 API 代价：borrowed `&DurableContext` 原型评估 94 文件迁移后推迟，运行时 check 兜底——"编译期拒绝" vs "运行时拒绝"的取舍未定论。
- AdminServer 无认证 by design（与 DBOS SDK 对齐）——安全边界完全依赖网络层，是否值得保留存疑。
