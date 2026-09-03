# Flow Atlas — deepseek-harness（七类流）

> Job: `ARCH-2026-09-03-001` | 每条流从真实代码导出，Edge 可回溯到 symbol / file / condition / state transition。
> 禁止根据架构想象画流程；每条 flow 的 chain 节点都带 `symbol`。
> **Reconciled**：F-05 Authority 顺序已按独立验证修正（collapse→pre-execute→approval→guards→execute）。

---

## F-01 · Control Flow — 谁决定下一步？
- **question**：Agent 回合中，谁决定进入哪个步骤/是否继续/何时终止？
- **chain**:
  1. `ReactLoopAgent.runLoop`（`packages/core/agent-loop/src/agent.ts:256`）— 产生回合（driver reservation 独占）
  2. `this.preStep(target, {turn,step})`（`agent.ts:283`）— 决策点：reject→blocked；空消息→completed（不消耗模型）
  3. `this.inbox.nextStep` 检查（`agent.ts:317`）— 决定进入 next-step 还是回合终止
  4. `agent/turn-stopping` waterfall（`agent.ts:319`）— 回合末通知监听者
  5. `agent/request-error` waterfall（`agent.ts:395`）— 请求失败时的重试/处理决策
- **gates**:
  - `signal.throwIfAborted()`（多处）— 取消门控
  - `if (this.phase.kind !== 'running') throw`（`agent.ts:259,343`）— 阶段守卫
- **states**: `idle / running(maintenance) / idle`（`agent.ts:39,72,102`）
- **evidence**: EV-001, EV-002, EV-025

## F-02 · State Flow — 状态如何变化？
- **question**：会话/回合/代理状态如何迁移？如何恢复？
- **chain**:
  1. phase 状态机：`setPhase(next)` 提交并发布外部可见状态迁移（`agent.ts:113`）
  2. `turn/start` append → `phase.turn = turn`（`agent.ts:262,268`）
  3. `step/start` append → `phase.step = step`（`agent.ts:289,294`）
  4. `step/end` / `turn/end` 在 `finally` 无条件追加（`agent.ts:305,356`）— 保证状态迁移可重放
  5. 恢复：`deriveMessages()` 从日志重建（`session/src/index.ts:790`）；projection `stateOf()` 增量折叠（session-projection）
- **conservation_points**: turn/step 事件对成对闭合（start/end 总是成对，即使 reject/abort 也写 turn/end）
- **states**: `idle→running→idle`；`turnEnds: completed|blocked|max-tokens|aborted|error`（`agent.ts:193-217`）
- **evidence**: EV-001, EV-003, EV-007

## F-03 · Data Flow — 数据从哪到哪？
- **question**：模型可见历史、提示词、LLM 输出如何流转？
- **chain**:
  1. `SessionEvent` 日志（`session/src/types.ts:261`）— 单一真相源（追加）
  2. `deriveMessages()`（`session/src/index.ts:790`）— 日志→消息数组（派生）
  3. `renderPrompt(assembly)`（`agent.ts:286`）→ `ctx.llm.stream(request)`（`agent.ts:330`）— 提示词→LLM
  4. `assistant/chunk` → `BlockAssembler.push`（`agent.ts:324-334`）— token 流→装配
  5. `assistant/message`（含 usage、interrupted 标记）→ 回到日志（`agent.ts:338`）
- **conservation_points**: 日志 lossless（原始 chunk 也保留，`chunkSeqs`）；usage 随 message 一起（无独立 usage 记录）
- **data_forms**: `SessionEvent[] → Message[] → StreamChunk → AssistantMessage`
- **evidence**: EV-003, EV-004

## F-04 · Evidence Flow — 声明如何变可证明？
- **question**：运行时的"事实"如何被断言/验证/归属？
- **chain**:
  1. session log（`session/types.ts:261`）— 权威事件流（唯一可断言源）
  2. `ctx.invariants`（`packages/runtime-diagnostics/invariants/src/index.ts`）— package-owned 检查，注册于各包 `./invariant` companion
  3. `InvariantFailure`（`invariants/src/index.ts`）— 违规即抛错（loud）
  4. 测试：vitest 835 spec + per-file 100% 覆盖率门禁 + invariant.spec.ts（每包）— 把不变量固化为测试证据
  5. postmortem 归属：结构化错误 code（TOOL_TIMEOUT/SANDBOX_UNAVAILABLE）供重放路由（`guard/timeout-policy`, `postmortem/0004`）
- **gates**: 断言范围受 AGENTS.md conventions 约束（只能断言权威事件流/可变数据，不得断言服务/方法存在）
- **evidence**: EV-037, EV-022, EV-013

## F-05 · Authority Flow — 每一步谁有执行权？[R：顺序已按独立验证修正]
- **question**：一次工具调用/文件写入/子进程，如何获得并受制于执行权？
- **chain**（真实顺序，代码实证 `tools/index.ts:1367,1400,1468-1493`）:
  1. 工具调用 → `tools/execute`（`core/tools/src/index.ts:227`）— 入口
  2. **collapse-check（确定性失败）**（`tools/index.ts:1367-1411`）— collapsed 调用在策略管线**之前**确定性拒绝（`ToolNotFoundError`，策略监听器不可见、不可批准注定失败的调用）
  3. `tools/pre-execute` waterfall（`tools/index.ts:144`）— pre-execute 可返回 `ask`（触发审批）或改写参数
  4. **approval**（`ctx.approval.request`，`interaction/user-approval/src/index.ts:207`）— Answerer 链，fail-closed `unavailable`；`never` 政策在 dispatch 前确定性拒绝（`index.ts:258-264`）；per-session override 经 `effectivePolicy = overrideOf ?? config.policy ?? 'ask'` 解析（`index.ts:29-35,236`）
  5. **guards（单调拒绝）**（`tools/index.ts:697`）— 仅当 decision.allow 时评估；guardReason 取第一个拒绝；守卫无 allow 语义
  6. `sandbox.confine`（`sandbox/sandbox/src/index.ts`）— per-call 模式（read-only/workspace-write/danger-full-access）约束子进程 argv
  7. `credentials.resolve`（`credentials/credentials/src/index.ts:2`）— 每操作重解析 env 引用
  8. `tools/post-execute`（`tools/index.ts:167`）— 结果层复核/改写
- **gates**: `ApprovalOutcome` 非 `allowed-once` 即拒绝；守卫只有拒绝无 allow；approval 先于 guards（非之后）；sandbox partial enforcement 作为上报事实
- **evidence**: EV-011..021 + IF-01（顺序修正）
- **flow_traceability**: 每条边可回溯实际 symbol；collapse→approval→guards 顺序与 `tools/index.ts:1367,1400,1468-1493` 一致

## F-06 · Memory Flow — 记忆如何沉淀？
- **question**：会话记忆如何从原始日志变成模型可用的上下文？
- **chain**:
  1. session log（事实层，append-only）→ `surface` 投影（模型可见层，`surface.ts:29-67`，surfaceOp append/replace）
  2. projection 增量折叠派生状态（session-projection，`stateOf()/snapshot()`）
  3. 上下文组装：`ctx.systemPrompt` 有序 section（`core/system-prompt/src/index.ts:2,52`）→ `renderPrompt`
  4. compaction：`compactNow` 替换历史范围为 summary 节点（`compaction/compaction/src/index.ts:120-162`）；`compaction/start` 持久事件
  5. 持久化：JSONL/zstd（`session-persistence-jsonl/src/format.ts:37`）或 SQLite（`storage-sqlite/src/schema.ts`）
- **gates**: 只有 idle agent 可压缩；history changed-span 竞态拒绝并记录；非 surface 事件永不在模型 transcript
- **evidence**: EV-004..007, EV-026..028

## F-07 · Policy Flow — 系统如何改变自己"下一步"的规则？
- **question**：一次性安全决策如何固化为持久策略，并约束未来的决策？
- **chain**（标准治理闭环）:
  1. **Decision**：用户/Answerer 对一次审批请求给出 allowed-once（`user-approval/src/index.ts:258`）
  2. **Approval**：判断生效为授权（`allowed-once` grant）；`approval/decided` 审计事件
  3. **Policy**：`setApprovalPolicy(session, policy)` 把 ask/never 写入 `approval/policy` 事件（`index.ts:92`）— 策略持久化到日志（重放可重建）
  4. **Enforcement**：后续 `request()` 读 `effectivePolicy(session)`（`index.ts:235`）— never → 确定性拒绝；sandbox mode / permission preset 约束后续执行
  5. **Future Decision**：策略改变未来判断（同会话后续工具调用免审/拒绝；danger-full-access+never 用于 unattended CI）
- **gates**: `setApprovalPolicy` 单一写路径；策略经日志重放重建（无旁路策略状态）
- **evidence**: EV-014..016, EV-021

---

## 重复结构观察（供 pattern-miner / synthesizer，非结论）
- 结构 `collapse→pre-execute→审批→守卫→执行→记录` 在 F-01/F-05/F-06 反复出现 → **Observed repeated structure**（已由 KO-03/KO-05 承载）
- 结构 `Decision→Approval→Policy→Enforcement→Future` 在 F-07 与权限升级/预设中重复 → **Observed repeated structure**（已由 KO-06 承载）
- 结构 `只追加日志 → 派生视图` 在 F-02/F-03/F-04/F-06 全部出现 → **Observed repeated structure**（已由 KO-01/KO-04/KO-05 承载）
