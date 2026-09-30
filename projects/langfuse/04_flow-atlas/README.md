# Flow Atlas — langfuse（七类流）

> 从真实代码导出。每条关键 Edge 标注回溯位置（file:行号 / symbol）。

## F1. Control Flow（控制流）
**eval 规则执行控制**
```
eval 规则(ACTIVE?) ── isEvalRuleExecutable ──► 执行 ──► 出结果
        │ blockedAt?                        │ MANUAL 绕过 toggle
        ▼                                    ▼
   不执行（blocked）                     不绕过 blocked
```
- Edge: `canRunEvalRule`（evalConfigBlocking.ts:41-50）：MANUAL 模式仅绕过 live-traffic toggle，blocked 规则永不可手动执行
- Edge: `isEvalRuleExecutable` = status===ACTIVE && !blockedAt（evalConfigBlocking.ts:35-39）

**ingestion 控制**
```
事件入队 ── getDelay(延迟决策) ──► worker 消费 ── isTraceIdInSample ──► 落库 / 丢弃
```
- Edge: getDelay（processEventBatch.ts:59-86）：跨日边界强制延迟；OTel 0；API min(5000,env)
- Edge: isTraceIdInSample（sampling.ts:5-23）：未配置/无效率 → 保守保留

## F2. State Flow（状态流）
**Eval 规则状态机**
```
[未创建] → ACTIVE ──(blockedAt 设置)──► blocked(PAUSED display)
              ▲                              │
              └──────── reactivate ─────────┘
```
- Edge: EvalExecutionMode z.enum(["LIVE","MANUAL"])（evalConfigBlocking.ts:30-32）
- Edge: PausedEvaluatorDisplayState = "PAUSED"（evalConfigBlocking.ts:6-8）：展示层状态与 JobConfigState 分离

**OTel 处理器内部状态**
```
TraceState { hasFullTrace: boolean, shallowEventIds: string[] }
```
- Edge: 处理器逐 span 更新 hasFullTrace（OtelIngestionProcessor.ts:57-80）：决定事件是"完整 trace"还是"浅事件"

## F3. Data Flow（数据流）
```
SDK/OTel ─► ingestion API ─► Redis queue ─► worker ─► (采样/匹配/归一化) ─► ClickHouse events
                                                          │
                                                          └─► S3（大事件溢出）
```
- Edge: processEventBatch（processEventBatch.ts:1-100）：批处理 + S3 上传
- Edge: eventsTableTraceNameSelectSql（eventsTable.ts:20-30）：wire '' ↔ JS null 归一化
- Edge: normalizeEventsTraceName（eventsTable.ts:38-43）：JS 消费面 null 化

## F4. Evidence Flow（证据流）
**eval → Score**
```
judge 输出 ── compilePersistedEvalOutputDefinition ──► validateEvalOutputResult ──► Score 写回
```
- Edge: compileEvalPrompt + CHAT_MESSAGE_BUILDERS（llmEvaluatorExecution.ts:1-50）：模板插值
- Edge: buildDecisionModelState（decisionModelEvaluatorExecution.ts:78-90）：null/undefined 跳过 → 干净 state
- Edge: DecisionModelClient.evaluate 返回 probabilities+confidence（decisionModelEvaluatorExecution.ts:41-60）

## F5. Authority Flow（权威流）
```
请求 ──► 认证（ApiKey / next-auth Session / AuthHeader 校验）──► RBAC（projectAccessRights）──► 授权资源
```
- Edge: AuthHeaderValidVerificationResultIngestion（auth/types.ts）：摄取认证结果类型化
- Edge: projectAccessRights.ts（rbac/）：项目级权限面
- Edge: Prisma ApiKey/Session/VerificationToken（schema.prisma）：认证数据模型

## F6. Memory Flow（记忆流）
**缓存**
```
modelMatch 查价 ──► LocalCache(L1, TTL-only 10s/20k) + Redis 失效信号 ──► 回源 DB
```
- Edge: modelMatchLocalCache（modelMatch.ts:31-40）："intentionally TTL-only; cross-container consistency from Redis invalidation"
- Edge: LOCK:model-match-clear（modelMatch.ts:29）：Redis 清缓存锁
- Edge: setNoEvalConfigsCache（evalService imports）：eval 配置负缓存

## F7. Policy Flow（策略流）
**供应链策略闭环**
```
决策（依赖准入策略）──► 执行（minimumReleaseAge/allowBuilds/overrides/patched）──► 验证（pnpm lockfile/CI）──► 未来决策（新依赖再次走 5 天窗口）
```
- Edge: minimumReleaseAge=7200（pnpm-workspace.yaml）：新依赖 5 天观察期
- Edge: trustPolicy no-downgrade：拒绝信任弱化
- Edge: allowBuilds 白名单：构建执行面收窄
- Edge: AGENTS.md 治理策略（subagent 委派/测试纪律/种子优先）：开发行为策略固化

---

## Flow→KO 交叉校验
- F2 blocked 状态机 ↔ KO-04（治理不变量）：验证通过（状态机实现即不变量）
- F6 缓存链 ↔ KO-07（双存储）：验证通过（缓存层是元数据查询的性能面）
- F7 供应链策略 ↔ KO-06：验证通过（四层策略在 pnpm-workspace 实际存在）
- F3 数据流 ↔ KO-01：验证通过（事件化管道 Edge 全部可回溯）
