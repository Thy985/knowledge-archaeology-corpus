# Infinity — Knowledge Archaeology Overview（ARCH-2026-10-29-001）

> 一句话定位：**"agent 即数据"的极低内存并发挂起 runtime**——把 agent 变成 step 间无进程、无栈、无连接的纯数据，挂起免费、事件驱动恢复、serverless 规模到零。Rust 实现，Apache-2.0，Hydro 项目（AWS × UC Berkeley）。

## 六条认知主线
1. **Slice 有界执行**：一步 = 加载 history → 一次模型补全 → 持久化 → 至多一个异步工具调用 → 结束；同步工具在同一 completion 内 loop-back（thread.rs:106-118, 500-620；event_processor.rs:1855-1900）
2. **Commit/Dispatcher 双屏障**：turn 持久化（`history.sync()`）先于工具执行/分派（async 工具在分派前、sync 工具在执行前，后者保证 spawn 状态正确）——调用在 history 中 durable 之前，其结果不可能到达；工具执行失败入队 error result 使 agent 恢复而非永久等待（thread.rs:579-606；event_processor.rs:1838,1960-1990）
3. **消息 ID 幂等去重**：processed_message_ids（内存 + DynamoDB String Set）拒绝重复投递；工具结果反向走查（Unanswered/AlreadyAnswered/NoPendingCall）判陈旧（event_processor.rs:244-330；state_store.rs:48-80）
4. **中断 = 合成事件**：用户文本打断 pending tool call → 注入合成 "interrupted" 结果保持 history well-formed；未验证输入不持久化，下次重发——不因中断丢 turn 也不 wedge（event_processor.rs:322-355, 468-560）
5. **Defer 语义平台契约**：DeferQueue——resident 可内存持有 synthetic events 等 tool 结果；serverless（Lambda NoDeferral）立即处理接受中断语义（defer.rs 全文件）
6. **Driver 生命周期与资源绑定**：Live/Idle 与 active_threads 同点报告防不一致；router 用 FuturesUnordered 池即时释放内存；订阅不保持 driver 驻留——事件 respawn（router.rs:1-8, 177-178；driver.rs:331-363）

## 关键数字（全部可回溯）
- 规模：139 个非测试 .rs / 48,484 行；测试 11,782 行（测试/代码 0.24x）；21 crates
- 核心：infinity-agent-core 15,050 行（29 文件）· infinity-daemon 7,490 · infinity-agent-cli 7,195 · sandbox-core 3,543 · infinity-agent-lambda 1,894 · rap-protocol 392
- 版本：Apache-2.0；sub-crate releases v0.2.0（10-22）；HEAD `1abc6fc`（10-27）
- 外部：rig-core 0.31 / AWS SDK（dynamodb/sqs/scheduler/dsql）/ sqlx 0.8（postgres）
- 治理：AGENTS.md（Conventional Commits + check.bash 门 + expect 纪律）；CI 4 workflows

## Corpus 连接
- **与 restate（10-27）/ dbos-transact-py（10-28）构成 durable execution 三路线对照**：restate=专用 bifrost 日志段；dbos=数据库即日志；infinity=并发挂起（agent 即数据，slice 卸载）。KO-D05/D08 跨项目验证待 10-30 品类横向
- **与 TeamMind（用户项目）**：actor-style 线程模型（spawn_thread fresh context / report_to_parent / 线程仅消息）+ 事件驱动 = 直接对照素材

## 审计声明
- Coverage：核心 runtime（agent-core）+ 云部署（lambda）+ 协议（rap）+ 沙箱（sandbox）已覆盖；daemon 深层 session 状态机、UI（infinity-ui/web）、Bedrock provider 细节为 PARTIAL
- 闭源/不可考古面：无（Apache-2.0 全开源）；但性能基准（75k/8GB）来自官方博客（S5 级），非本仓库实测
- 质量指标：24 EK（EK Graph）· 8 KO（R1×2/R2×2/R3×1/R4×3；L3×2/L4×5/L5×1）· 34 Flow Edge · 7 Candidates · 采证锚 45+
