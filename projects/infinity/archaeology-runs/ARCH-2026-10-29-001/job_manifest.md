# Job Manifest — ARCH-2026-10-29-001

> 依据：KnowlegeMap HEAD `9c0a467`（2026-10-29 radar watch：1 张 B 级新卡 AWS Infinity）+ 品类卡 `[cand]aws-infinity-agent-runtime-2026-10.md`（10-29 建卡）+ corpus 已有状态（restate 10-27 / dbos-transact-py 10-28 已考古）。

## 候选排序（按 Knowledge Value 优先，不按 star）
| 排名 | 候选 | 类型 | 状态 | 理由（Knowledge Value/相关性/新颖度/连接） |
|---|---|---|---|---|
| **1** | **hydro-project/infinity** | NEW · P0 · initial | **选中** | 第三条 agent runtime 设计路线（stackless coroutine 挂起/slice/step 模式）；10-23 刚开源 Novetly 最高；与 TeamMind 并发模型直接对照；与 corpus restate（专用日志段）/dbos（数据库即日志）构成三路线对照，支撑 10-30 周五验证日；Benchmark 学习价值高（事件驱动恢复/幂等去重） |
| 2 | Temporal（durable 品类 7-way 缺口） | QUEUED | 否决 | 与 restate/dbos 同思路（步骤持久化 checkpoint），品类重复度高，边际递减；留 10-30 横向 |
| 3 | Dapr Agents / Mastra / MS Agent Framework | QUEUED | 否决 | 品类卡已覆盖/去重；增量不新 |
| 4 | Vercel durable-harness / Azure bindings / Pi Durable / Cloudflare SDK | C 级 | 否决 | 品类卡去重命中（低于建卡阈值） |

## 选中项目
- **project**: infinity
- **repository**: https://github.com/hydro-project/infinity.git
- **mode**: initial
- **priority**: P0（B 级新卡）
- **reason**: Agent Runtime 深耕领域第三条设计路线——极低内存并发挂起（slice + serverless 卸载），与 durable checkpointing 路线互补；品类卡证据完备（官方博客 FACT ×3：Apache 2.0 开源/75k 并发 8GB/step 模式零计算）；10-22 v0.2.0 系列 release 活跃；corpus 无相关考古
- **source_entry**: `03_expansion_queue/candidates/[cand]aws-infinity-agent-runtime-2026-10.md`（2026-10-29 radar watch Day4 增量建卡）
- **元数据**: Rust · Apache-2.0 · 207★/23 forks · created 2026-04-13 · pushed 2026-10-27 · default main · 4.9MB · topics=agents/concurrency/distributed-systems/rust · releases=sandbox-remote/sandbox-local/sandbox-core/rap-steering-server/rap-protocol v0.2.0（10-22）

## 预期考古焦点
1. slice 生命周期（加载 history→模型补全→持久化→至多一个工具调用→结束）
2. step 模式（serverless 卸载、at-least-once + 消息 ID 幂等去重、FIFO 组串行化免锁）
3. synthetic tool calls（receive_event__injected）+ 订阅事件/associative 事件分派
4. spawn_thread/report_to_parent 父子线程协议（无需独立消息系统）
5. 并发密度（8GB/75k）实现机制（history 内存占用 vs 进程占用）
