# 05 · Candidates — AWS Infinity

> 未验证假设 / 跨项目候选 / 不确定结论。严格区分于 03 的 KO（Candidates 是 Hypothesis，不冒充知识）。

## C-01 · 挂起-恢复 vs 步骤重放的资源经济学对比（跨项目假说）
- **假设**：挂起-恢复派（Infinity）在"长空闲、低频唤醒"场景总成本低于步骤重放派（Temporal/Restate/DBOS），因为空闲成本≈历史存储，而重放派空闲也需进程/worker 常驻。
- **当前证据**：Infinity 154 KB/agent（S4，CHANGELOG 自述）；Temporal/Restate 无同口径数字（corpus 已考古，但未直接对比）。
- **缺失证据**：同 workload 双实现对照 benchmark；唤醒延迟对比（Lambda 冷启动 vs 重放恢复延迟）。
- **验证路径**：双派同构 workload（如 10k 长空闲 agent + 突发唤醒）在 AWS/自建环境复测。

## C-02 · slice 语义的通用性（跨项目候选 Pattern）
- **假设**："每 slice 至多一次补全 + 至多一次工具派发 + 完全退出"是通用 serverless agent 执行模板（不只是 Infinity）。
- **当前证据**：Infinity 内部一致性高（step-mode 文档 + Lambda 实现 + NoDeferral）；AWS 官方博客宣称可移植。
- **缺失证据**：其他框架显式采用同一 slice 结构的证据；NoDeferral 中断语义在真实高延迟工具下的表现。
- **验证路径**：调研 MCP/AG-UI/其他 agent runtime 是否有等价 slice 结构；或复刻一个最小 slice runtime 跑基准。

## C-03 · 幂等去重的可移植边界
- **假设**："用户文本/事件按 ID 去重 + 工具结果按 history tail 去重"策略可安全移植到其他 durable execution 框架。
- **当前证据**：Infinity 双生产实现（Lambda/daemon）一致；测试覆盖 stale result 场景。
- **缺失证据**：跨框架（如 DBOS 的 workflow 幂等）对比；多 writer 并发写同一 history 的边界（Infinity 是 per-thread 单写者）。
- **验证路径**：在多写者场景做故障注入测试。

## C-04 · compaction 与 spawn 交错（Infinity 内部未决）
- **观察**：compaction×child 修复史长（4+ 提交：be6bbd5/d27a405/a536c9f/cee7fa7）——该区域是系统最脆弱处。
- **当前证据**：4 个专项测试覆盖；cee7fa7 引入"seed children with own spawn call"。
- **缺口**：compaction 与"fresh context 边界"交错未见专项测试；多代子孙链 + 多次 compaction 的组合未见。
- **验证路径**：审计 cee7fa7 后的测试覆盖矩阵；补嵌套多代 compaction 压力测试。

## C-05 · 中断语义的模型层面影响（跨项目假说）
- **假设**：频繁 synthetic "interrupted by user" 注入会轻微毒化模型对工具结果的信任（模型学到"结果可能被中断"），长期影响工具使用质量。
- **当前证据**：`TOOL_CALL_INTERRUPTED_TEXT` 明确区分"interrupted"与"too large"placeholder（设计者已考虑误导风险）。
- **缺失证据**：量化评估（A/B 提示词 + 工具使用成功率）。
- **验证路径**：在控制实验中对比"中断注入频率"与"工具调用正确率"。

## C-06 · 订阅事件的 child 处理成本
- **观察**：非 associative 订阅事件每次 spawn 临时 child——高事件率订阅（如流式日志）会放大上下文复制成本。
- **当前证据**：文档明示 inline（associative）适合"需要看到更新的场景"，threaded 默认适合隔离；权衡未量化。
- **缺失证据**：高事件率下两种策略的 token/延迟/成本实测。
- **验证路径**：同事件流跑 inline vs threaded，对比 token 消耗与延迟。

## C-07 · RAP 协议 vs MCP 生态（Cross-project Candidate）
- **观察**：RAP 提供 MCP 兼容层而非 fork，定位为 MCP 的异步扩展。
- **假设**：异步工具协议（RAP 类）将成为 agent 工具生态主流形态，MCP 同步模型逐步让位。
- **当前证据**：Infinity 设计文档立场 + bridge 实现；工具生态趋势（外部观察，非本项目证据）。
- **缺失证据**：RAP 采用率数据；协议迁移案例。
- **验证路径**：跟踪 RAP spec 在 Hydro 项目外的采用。

## C-08 · 编译期线程安全约束的缺失（内部观察）
- **观察**：`ThreadObserver::on_event` 同步调用 + `?Send` trait 标记（async_trait(?Send)）——observer 在 slice 的异步边界内同步执行，长耗时 observer 会拖慢 slice。
- **当前证据**：文档明示 "keep it fast"；?Send 放宽约束（可能牺牲并发安全）。
- **缺失证据**：observer 长耗时对吞吐的实测影响。
- **验证路径**：压力测试 + 慢 observer 注入。

## C-09 · "154 KB/agent" 的适用边界
- **观察**：benchmark 为 in-memory stores + 脚本化模型 + 单线程；真实场景（DSQL/DynamoDB + 真实 Bedrock + 多线程）数字不同。
- **当前证据**：CHANGELOG 方法学（VmRSS after malloc_trim，idle 采样）透明。
- **缺失证据**：云存储版本的内存/成本对比。
- **验证路径**：Lambda 部署 + 同 workload 实测。
