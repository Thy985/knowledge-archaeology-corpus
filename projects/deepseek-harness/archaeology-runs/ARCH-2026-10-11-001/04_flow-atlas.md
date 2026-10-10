# 04 Flow Atlas — deepseek-harness v0.2.1-alpha.2（七类流）

> 从真实代码/文档导出，关键 Edge 可回溯 symbol/file/condition/state transition。STABLE 流标注基线维持，NEW 标注 0.2 新增 edge。

## 1. Control Flow（控制流）

```
dsh --profile <name>（唯一启动入口）
  → boot 组装插件树（profile bundles → cordis.patch.yml → home patch → --patch overlay）      [EK-27]
  → AgentLoop 驱动：turn/start
      → claim next-step input + 1 queued message                                          [EK-06]
      → agent/pre-step（rewrite | reject → 无 step 关闭 turn；enter → startsRequestSeries）[EK-07]
      → step/start → agent/request → prepareCall（取消 → 提交 neither）                    [EK-08]
      → 冻结模型历史 → llm/stream → assistant/message | assistant/attempt                 [EK-04]
      → tool/call* → tools/pre-execute → execute → post-execute → tool/result*             [EK-13]
      → guard 干预：repeat-tool-reminder（post-execute 决策 enrich）/ timeout-policy（deadline 到点报错）[EK-12]
      → step/end → 是否再 claim（tools owe / next input）→ agent/turn-stopping → turn/end
```
关键 edge：`agent/pre-step` 拒绝/空 → 无 step turn（`reject, or a first enter rewritten empty -> close the turn with no step`，docs/architecture.md）；`agent/request`→`prepareCall()` 取消不提交（cancellation commits neither system nor users）。

## 2. State Flow（状态流）

```
SessionEventMap（append-only 事实源）                                            [EK-01]
  → deriveMessages() 投影模型历史（model-visible means logged）                    [EK-01]
  → dsh-session-projection：注册单元增量折叠 → stateOf() / snapshot()             [EK-11]
  → 持久化：JSONL v0 session.jsonl[.zstd] / v1+ session.vN.jsonl[.zstd]           [EK-31]
  → SESSION_FORMAT_VERSION=4 世代选择（stat/list 选最高 canonical generation）     [EK-02]
  → 迁移：adjacent vN→vN+1（静态链一次编译；write open 独占发布 successor）        [EK-03]
Agent 状态：idle|running（agent/* 事件：inbox/step/status/request/validation/continuation）[EK-09]
```
新增状态节点（NEW）：goal phase（pending/active/paused/completed/blocked，CAS revision）、job status（running/stopping/completed/killed/failed）、schedule task（active/inactive + nextUTC scheduledAt）。证据：packages/goal/goal/src/fold.ts、packages/jobs/jobs/README.md、packages/schedule/schedule/README.md。

## 3. Data Flow（数据流）

```
模型请求 → prompt sections + tool schemas 组装（systemPrompt）                    [EK-10]
  → 工具执行：ctx.tools 作用域注册表 → 把关（policy pre/post + guard）             [EK-13]
  → bypass edge（Reconciliation I-15）：pipeline failures that bypass post-execute skip projection —— 管道失败绕过 post-execute 时跳过投影（tools/src/index.ts:239-242）
  → 超大结果：spill 保存全文 → 模型只见 bounded preview + locator（maxInlineTokens）[EK-24]
  → turn 交付：present 工具声明交付文件；workspace-changes 记录 git 快照行数       [EK-23]
  → PTC：model-written program → ctx.ptcRuntime.resolve/run → lossless-JSON result/logs/error [EK-14]
  → 后台：jobs output ring（stdout/stderr→model、log→observers only）             [EK-21]
```

## 4. Evidence Flow（证据流）

```
session log = 唯一真相源（append-only）
  → assistant/message 内嵌精确紧凑流（重建 assembled content）                     [EK-04]
  → assistant/attempt 保留 settled 失败/重试/取消/流错误（不加模型历史）            [EK-04]
  → failed steps 记录 missing tool results（不伪装成功）                           [EK-05]
  → 可重放：fork/resume/transcripts/telemetry 全从 durable settlements 派生       [EK-01]
```
关键 edge：`model-visible means logged`——新模型可见输入必须产生 session 事件（docs/architecture.md §Session log）。

## 5. Authority Flow（权威流）NEW 强化

```
SAFETY.md 声明：沙箱/审批/权限不保证隔离；不做 untrusted workload 唯一安全控制     [EK-16]
  → ctx.sandbox 后端（sandbox/sandbox-local/sandbox-policy/sandbox-windows-acl）包装 argv [EK-15]
  → approval / permission-presets / credentials seam                              [EK-17]
  → PTC isolation 描述符：诊断信息，不承诺安全边界                                [EK-14]
  → 应用启动：verify-application-entrypoints 拒绝绕过 dsh 的路径                  [EK-18]
  → 动态扩展：no model tool creates dynamic definitions（进程级 node:vm）          [EK-19]
```
关键 edge：extensions 动态定义**重启消失**（Definitions disappear on restart，cordis-host-runner README）；持久安装唯一通道 Plugin Manager。

## 6. Memory Flow（记忆流）NEW 控制面

```
session log（事实记忆）→ projection（读取视图）→ compaction seam（上下文压缩）     [EK-01/11]
  → goal：同会话持久目标（1/session，round cap 256，CAS，continuation 权限进程级）  [EK-20]
  → schedule：host-wide durable reminders（delivery 需 Host controller + flush 确认）[EK-22]
  → jobs：后台任务结果 ring（owner 会话隔离，settlement notice 免轮询）            [EK-21]
  → spill：超大文本离线保存（opaque locator）                                     [EK-24]
```

## 7. Policy Flow（策略流）NEW

```
决策（design/architecture 文档化：system-prompt-as-surface-node / session-projection-mandatory-seam / released-session-format-migrations）
  → 策略固化（AGENTS.md 治理纪律 + docs/config-catalog.md 生成式配置）
  → 执行（guard 策略：repeat-tool/timeout；sandbox-policy；approval；permission-presets）
  → 未来决策（released-format obligations 触发版本升级决策；SAFETY.md 约束部署决策）
```
关键 edge（外部要求，见 C-02）：10-11 雷达"评测隔离（air-gapped eval）成为厂商标配"要求 eval harness 默认 fail-closed 网络策略——dsh 网络策略默认配置未实测（Candidate）。

## Flow→KO 交叉校验

| KO | 依赖流 | 校验 |
|---|---|---|
| KO-02 | Control Flow（pre-step→prepareCall→提交）| 一致（docs §Turn flow）|
| KO-03 | State/Evidence Flow（log→projection→版本化）| 一致（session-format-status + types.ts）|
| KO-04 | Authority Flow（SAFETY→sandbox→PTC）| 一致（SAFETY.md 原文）|
| KO-05 | Memory Flow（goal/jobs/schedule/deliverables/spill）| 一致（五包 README + fold.ts 实现）|
| KO-07 | Authority Flow（启动纪律）| 一致（verify-application-entrypoints.ts）|
| KO-08 | State Flow（版本化迁移）| 一致（session-format-status.md）|
| KO-09 | Control Flow（guard 干预）| 一致（guard src）|
