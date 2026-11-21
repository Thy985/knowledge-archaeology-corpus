# 06 · Validation & Evidence — AWS Infinity

> 六 Auditor 先 Blind Reconstruction（独立重读仓库建立独立发现），再对比本文档。
> 本文件 = Truth / Coverage / Flow / Abstraction / Counterexample / Epistemic 六视角验证记录 + 反例 + 证据强度。

## 1. Truth Auditor（这句话是真的吗？）

Blind Reconstruction 独立重读的关键断言（不读本 package 前提下重读仓库）：

| # | 断言 | 独立重读验证 | 判定 |
|---|---|---|---|
| T-1 | slice 三阶段函数存在于 event_processor.rs | 独立 grep 确认 `prepare_input`(889)/`run_completion`(1480)/`execute_action`(1933) | ✓ CONFIRMED |
| T-2 | `spawn_thread` 是同步工具（execute_synchronous） | 独立确认 tools/thread.rs:65 `async fn execute_synchronous` | ✓ CONFIRMED |
| T-3 | 工具结果无需 durable ID（history tail 去重） | traits.rs StateStore 注释原文确认 | ✓ CONFIRMED |
| T-4 | compaction >75% context 触发 | driver.rs:384-385 原文 `> 75% full` | ✓ CONFIRMED |
| T-5 | Lambda 用 `NoDeferral` + 同队列回环 | event_handler.rs 原文 + SqsMessageSender | ✓ CONFIRMED |
| T-6 | benchmark "50k×20=2M completions 255s 7.9GB" | CHANGELOG 原文 + agent_scale.rs 方法学 | ✓ CONFIRMED |
| T-7 | ChoiceId 目前是 ToolCallId 别名 | rap-protocol lib.rs（issue #109 跟踪） | ✓ CONFIRMED |
| T-8 | 非 associative 订阅事件默认 spawn 临时 child | subscription-events.md "Threaded processing (the default)" | ✓ CONFIRMED |

**计数类断言（独立重数）**：
- Rust 文件数 168 / 57,187 行 —— 独立 python3 重数 ✓（本次考古以脚本统计为准）
- 21 crates —— Cargo.toml workspace members 独立计数 ✓
- tests.rs 28 个测试函数 —— 独立 grep 计数 ✓
- .snap 快照 150 个 —— 独立 find 计数 ✓

## 2. Coverage Auditor（还有什么重要东西没发现？）

独立重搜 + 强制子系统覆盖：

| 子系统 | 覆盖 | 补充发现 |
|---|---|---|
| agent-core 循环 | 完整（slice/history/thread） | 无遗漏 |
| provider 层 | provider-protocol 已覆盖；provider-rig 未深挖（504 行，rig 适配器） | **缺口**：rig 适配器的具体差异未单列 EK |
| sandbox | ModeProvider 机制已覆盖；jj/git 具体实现细节未深挖 | **缺口**：sandbox-remote（534 行）网络隔离实现未验证 |
| slack bot | 仅提及（2,371 行） | **缺口**：dataflow 架构未考古 |
| UI/TUI | 修复史已覆盖（ANSI/wide char/scrollback）；内部组件未考古 | 属于产品层，非运行时核心 |
| 协议 | RAP spec 已覆盖；MCP bridge 细节未深挖 | **缺口**：bridge 的协议转换边界 |
| daemon session 管理 | 持久化/清理已覆盖；WS handler 细节未深挖 | 产品层 |

**结论**：核心运行时覆盖充分；产品/适配器层有 4 个已知缺口（不阻塞核心知识，列入 Candidates 或后续 refresh）。

## 3. Flow Auditor（Flow Atlas 是否真的对应代码？）

| Flow Edge | 代码锚点 | 判定 |
|---|---|---|
| slice 三阶段 | event_processor.rs 三函数签名 | ✓ |
| sync 先于 dispatch | history-manager.md "a tool call is dispatched only after the turn that produced it has been synced" + execute_action 时序 | ✓ |
| FIFO per group | InputSender 契约 + SQS MessageGroupId 注释 | ✓ |
| spawn=新 group | thread.rs spawn_thread → conversation_store.spawn_thread → seed → send_to_input_queue | ✓ |
| synthetic 注入 | input_event()（event_processor.rs:1309）合成 AgentEvent；find_tool_call resolve | ✓ |
| deferral | DeferQueue push/drain + NoDeferral 返回 false | ✓ |
| compaction 摘要按 index | load_history_with_ancestors 默认实现 | ✓ |
| fallback/exception 路径 | `unless_closed` / `find_tool_call` 折叠进 summary 的 fallback（display 回退到 call ID） | ✓ |

## 4. Abstraction Auditor（L3/L4 是否过度升维？）

| KO | 升维审查 | 判定 |
|---|---|---|
| KO-01（挂起-恢复模型） | 单一项目（Infinity）→ 已标 Cross-project validation pending；内部证据充分（S4） | ✓ 保留 L4 |
| KO-02（幂等+FIFO 无锁） | 通用性有 actor 模型对照；但"无需锁"断言仅本项目验证 | ✓ 保留 L4，注 pending |
| KO-03（synthetic 注入） | 机制独特且自洽；通用性未跨项目验证 | ✓ 保留 L3 |
| KO-04（context 治理） | 与 RAG/长上下文文献一致；本项目测试充分 | ✓ 保留 L4 |
| KO-05（4-trait 可移植） | DBOS/step functions 有类似抽象（corpus 对照） | ✓ 保留 L3 |
| KO-06（睡眠一等公民） | 事件驱动 agent 通用 | ✓ 保留 L3 |
| KO-07（错误分类优先） | 网关类系统通用模式 | ✓ 保留 L3 |
| KO-08（测试=竞态纪律） | 方法学普遍；本项目证据强 | ✓ 保留 L4 |
| KO-09（工具=并发边界） | 新颖视角；需更多实现支撑 | ✓ 保留 L4，注 pending |

无降级需求；无 L5（方法论需跨项目）。

## 5. Counterexample Hunter（反例预算制）

每个 L3+ KO ≥3 个定向反例攻击：

| KO | 反例 1 | 反例 2 | 反例 3 | 结果 |
|---|---|---|---|---|
| KO-01 | NoDeferral 中断 pending call（slice 不变量≠turn 不变量） | 冷启动延迟 vs 重放派 | 状态全在历史，重放需重算 | 已内化（EK-16 标注） |
| KO-02 | 工具结果无 durable ID（tail 去重序敏感） | 多 writer 同 history 未支持 | FIFO 依赖传输层保证（SQS 语义） | 已内化（测试 stale_result） |
| KO-03 | `__injected` 模型可能学会调用 | 高事件率放大上下文 | 临时 child 增加 spawn 成本 | 部分内化（C-05/C-06） |
| KO-04 | compaction 摘要丢失细节（不可逆） | 大消息占位符误导模型 | 75% 阈值策略未自适应 | C-04 标注 |
| KO-05 | NoDeferral 暴露平台差异 | 4-trait 需平台正确实现 | 存储协议差异（DSQL vs JSON） | 已内化（EK-22/23/24） |
| KO-06 | sleep 中断结果追加次序 | 无 timer 平台的降级 | 事件风暴唤醒 | 已内化（EK-20/27） |
| KO-07 | provider 分类错误→错误恢复 | ContextOverflow 重试无界 | 分类契约质量依赖 provider | 已内化（C-07 未覆盖，标 open） |
| KO-08 | 28 测试无法覆盖全交错 | 快照测试维护成本 | 竞态测试 flaky 风险 | 已内化 |
| KO-09 | 同步工具 loop-back 增加 slice 长度 | 微秒级工具也走协议往返 | 并行工具调用上限未量化 | 已内化 |

## 6. Epistemic Auditor（认知状态诚实性）

| 检查项 | 状态 |
|---|---|
| Hypothesis 是否进 Candidates 而非 KO？ | ✓ 9 个 Hypothesis 全在 05，未冒充 KO |
| KO 是否都有 S4 级证据？ | ✓ 全部（实现+测试）；无 S5（无 Docker 声明） |
| Principle/Law 是否被误标？ | ✓ 无；全部标 Cross-project validation pending |
| 项目内经验是否写成普遍定律？ | ✓ 无；L4 均有 pending 标注 |
| design intent vs implementation 是否混淆？ | ✓ 已分离（doc 层 vs code 层证据分别标注） |
| 冲突证据是否保留？ | ✓（deferral 中断语义 vs slice 不变量；见 Counterexample 表） |

## 7. 证据强度分布

| 强度 | 含义 | 数量（主要） |
|---|---|---|
| S3 | Implemented | 源码引用（traits/event_processor/tools/lambda） |
| S4 | Test Validated | tests.rs 28 测试 + 150 snapshots + 4 compaction 测试 |
| S5 | Runtime Validated | **0**（无 Docker 环境，明确不升） |
| S8 | Cross-project | 0（全部标 pending） |

## 8. 反例与 Contradictions 汇总（保留在记录中）

1. **deferral 中断语义 vs slice 不变量**：EK-02 说"slice 永不阻塞外部"，但 NoDeferral 会中断 pending tool call——矛盾已通过"per-slice vs per-turn"区分消解（不变量按 slice 定义，中断按 turn 定义）。
2. **compaction 不可逆 vs 无损底座原则**：compaction 摘要丢失细节，与"EK 无损底座"哲学张力——已标 C-04（内部未决）。
3. **README "75k agents" vs CHANGELOG "50k measured"**：README 宣传 75k（commit 9c07368 "seventy thousand agents in 8 GB"），CHANGELOG 实测 50k×20turns 7.9GB——宣传数字与测量数字并存，标注为项目口径差异。

## 9. Validation 总结

- **CONFIRMED**：核心运行时全部关键断言（slice/FIFO/幂等/synthetic/compaction/Lambda 结构）
- **PARTIALLY_CONFIRMED**：订阅路由细节（threaded 默认路径代码较文档少）、sandbox remote 隔离
- **DOWNGRADED**：无
- **OVER_GENERALIZED**：无（全部 KO 已标 pending）
- **MISSING**：provider-rig 细节 / sandbox-remote 网络隔离 / MCP bridge 协议边界 / slack dataflow（4 项产品层缺口）
- **CONTRADICTED**：无直接矛盾；1 处宣传/实测数字差异（README vs CHANGELOG）
- **NEEDS_HUMAN_REVIEW**：C-02（slice 语义通用性）、C-05（中断注入的模型毒化）需领域专家判断

## 10. 质量指标

| 指标 | 值 |
|---|---|
| EK 总数 | 45 |
| EK 平均出边 | ≥2（全部有 links） |
| 游离 EK | 0 |
| KO 总数 | 9（L3×4 / L4×5） |
| KO 聚合覆盖率 | 100% |
| Candidates | 9 |
| 测试覆盖子系统 | 运行时核心全（28 测试 + 150 snapshots） |
| 证据上限 | S4（静态考古声明） |
| 反例攻击 | 9 KO × ≥3 = 27+ 攻击点 |
