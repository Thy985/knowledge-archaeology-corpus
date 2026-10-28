# Independent Validation Report — ARCH-2026-10-29-001

> Auditor 盲重建：独立重读仓库核心文件（不把主考古当事实源）后对比判定。禁止修改原考古产物；修正经 Reconciliation 写入 corpus。

## 判定统计
| 判定 | 数量 | 项 |
|---|---|---|
| CONFIRMED | 14 | IV-1..IV-14（见下） |
| CONTRADICTED | 0 | — |
| PARTIALLY_CONFIRMED | 1 | IV-15（"双屏障"表述对 async 工具严格成立） |
| DOWNGRADED | 0 | KO-D05 已标 Cross-project pending，风险已缓解 |
| MISSING | 5 | daemon session 深层 / UI（infinity-ui/web）/ Bedrock provider 细节 / default_prompt.md / sandbox git-jj 机制 |
| NEEDS_HUMAN_REVIEW | 0 | — |

## 14 CONFIRMED（重点）
1. **slice 有界执行**：thread.rs:106-118（step 间无 in-flight）+ 500-620 全流程
2. **commit barrier（async 工具）**：thread.rs:579-590 "durable in history before its result can possibly arrive"
3. **sync 工具同样先持久化**：event_processor.rs:1838 "we must sync the history so that thread spawning uses the correct state"（补充：commit barrier 对 sync 工具同样成立）
4. **dispatch 失败入队 error result**：event_processor.rs:1960-1990 + 测试 failed_tool_dispatch_returns_error_result_to_agent（system/tests.rs）——S4
5. **至多一个 terminal action 不变量**：thread.rs:659 `assert!(action.is_none(), "bug: multiple terminal actions")`——多 async tool call 是设计禁止（补充）
6. **消息 ID 幂等**：event_processor.rs:244-289（processed_message_ids 检查 + 立即 insert）
7. **match_tool_result 反向走查**：event_processor.rs:300-330（Unanswered/AlreadyAnswered/NoPendingCall，turn boundary 前全部结算）
8. **中断合成 interrupted**：event_processor.rs:322-355（`{id}-interrupted` 消息 ID）
9. **未验证输入不持久化 + drop 遗忘 dedup ID**：event_processor.rs:468-560（drop_unvalidated_user_inputs 中 processed.remove）
10. **DeferQueue/NoDeferral 语义**：defer.rs 全文件 + filter_deferrable（thread.rs:248-300，settles_pending 判断）
11. **driver Live/Idle 同点报告**：driver.rs:90-130（"channel and the set can never disagree"）
12. **check_before_idle 三态**：driver.rs:331-363（Continue/AwaitInput/Exit；订阅不驻留）
13. **Lambda 单 delivery=单 slice + group partition**：event_handler.rs:120-128, 174-178
14. **spawn_thread fresh context + safe_spawn_point**：tools/thread.rs:112-176（FRESH context 不继承 + safe_spawn_point 守卫）

## 1 PARTIALLY_CONFIRMED
- IV-15："commit/dispatch 双屏障"表述对 async 工具严格成立；sync 工具是"先 sync 再执行"（同一屏障、不同时机）。主考古 00 表述可接受，EK-D02 需补 sync 工具分支。

## 3 计数类复核（攻击目标，全部通过）
1. EK 平均出边 2.75：python 重算 24 EK 共 66 边 → 2.75 ✓（游离 0）
2. Flow 34 Edge：grep `^| (CF|SF|DF|EF|AF|MF|PF)-` = 34 ✓（Control 5/State 5/Data 5/Evidence 4/Authority 5/Memory 5/Policy 5）
3. crate 数 21：Cargo.toml members 逐一数 = 21 ✓

## 补充发现（Observation，Reconciliation 合入）
- **未知工具**：`{id}-unknown-tool` 合成错误结果 + loop back（event_processor.rs:1824-1835）——失败恢复第三条路径
- **Throttled 重试**：MAX_COMPLETION_RETRIES + THROTTLED_RETRY_DELAY（event_processor.rs:1558-1565）
- **取消语义**：initiation 阶段取消 → 本轮输入留内存下次重发、不持久化（event_processor.rs:1539-1544）
- **filter_deferrable settles_pending**：batch 含实际 tool result 或用户文本时 drain defer 队列（thread.rs:248-300）——defer 与中断的精确分界

## 遗漏（MISSING，主考古 Coverage=PARTIAL 声明确认）
- daemon session 状态机深层（session/{display,observer,mod}.rs）
- infinity-ui / infinity-web（前端）
- infinity-provider-bedrock 细节（60s initiation timeout、ErrorClass::Transient）
- default_prompt.md（agent 系统提示内容）
- sandbox git/jj 机制（stacked sandboxes 细节）

## Benchmark case（供 skill CI）
**Case**: "serverless agent 在等待工具结果时如何做到零计算占用？"
- 期望：agent 状态持久化 → 进程退出 → 事件到达 → 从持久化状态恢复
- Infinity 证据：slice（thread.rs:106-118）+ SQS 单 delivery 单 slice（event_handler.rs:174-178）+ NoDeferral（defer.rs）+ DynamoDB 幂等（state_store.rs:48-80）+ EventBridge timers（sleep.rs:42-122）
- 反例面：pending 工具期间 synthetic 事件 → NoDeferral 接受中断语义；多 terminal action → assert panic（不允许）
- 对照：restate（journal 重放）/ dbos（数据库 checkpoint）——三种"等待零占用"实现路径
