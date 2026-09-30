# Validation & Evidence — langfuse

## 1. 验证方法
- **Blind Reconstruction**：独立 Auditor（不读考古结论）重读 `repo/` 关键路径建立 Independent Findings，与本包对比（阶段⑤，独立报告见 independent-audit/report.md）。
- **Truth Audit**：每条 EK 的 file:行号 可回溯至克隆工作区实际内容。
- **Coverage Audit**：强制子系统覆盖——ingestion / otel / evals / worker / data / supply-chain / agent-governance 七面全过。
- **Flow Audit**：F1-F7 每条 Edge 标注 symbol/file。
- **Abstraction Audit**：KO-01~08 均带反例攻击与 Epistemic 状态；L5 方法论显式标 Cross-project validation pending。
- **Counterexample Hunt**：每个 KO ≥1 定向反例（见各 KO 的"反例攻击"节）。

## 2. 质量指标
| 指标 | 值 |
|---|---|
| EK 总数 | 48（44 + Reconciliation 新增 EK-45~48） |
| links 覆盖率 | 100%（48/48） |
| 平均出边 | 1.63（78 links / 48 EK） |
| 游离 EK | 0 |
| KO 总数 | 8（L4×6 / L3×1 / L5×1） |
| KO 聚合规则覆盖 | 100%（R1×2 / R2×2 / R3×3 / R4×2） |
| KO 平均簇规模 | 6.1（EK 口径） |
| Candidates | 6（C-01 关闭→转实证；C-04/C-05 更新） |
| 三层配比 | 48 EK → 8 KO（大项目按比例缩放，未硬凑 40-60/7-12 上限） |
| 证据可回溯性 | 全部 EK 带 file 引用；关键机制带行号 |

## 3. Validation 结果（先盲重建后对比）
- **判定统计**：CONFIRMED=34 / PARTIALLY_CONFIRMED=7 / DOWNGRADED=1 / OVER_GENERALIZED=1 / MISSING=1 / CONTRADICTED=0 / NEEDS_HUMAN_REVIEW=0（原包，见独立报告）+ 盲重建独立判定 9 项（PARTIALLY 5 / DOWNGRADED 1 / OVER_GENERALIZED 1 / MISSING 2）
- **成功 3 例**：
  1. EK-15 DecisionModel 结构化问题规格（CHOICE/SCORE/NOUL + limits）——独立重读 decisionModel.ts 确认
  2. EK-19 EvalRule 治理门（MANUAL 绕过 toggle 不绕过 blocked）——独立重读 evalConfigBlocking.ts:41-50 确认
  3. EK-40 pnpm 5 天延迟 + no-downgrade——独立重读 pnpm-workspace.yaml 确认
- **错误 3 例**：
  1. EK-28 原写"Secondary 队列=消费隔离" → 独立验证发现注释未证语义，降级为"存在 Secondary 变体"（PARTIALLY_CONFIRMED，见 C-01）
  2. EK-24 原写"eval 输出有 schema 不依赖自然语言解析" → 独立验证发现 judge 输出仍有自然语言路径（compileEvalPrompt 插值），收紧为"输出定义编译+校验"（DOWNGRADED）
  3. EK-14 原写"dogfooding 模式" → 独立验证仅见 internalTrace 写入器，未见消费闭环证据，降为"平台自身操作写入内部 trace"（PARTIALLY_CONFIRMED）
- **遗漏 1 例**：MISSING——独立 Auditor 发现 `evalExecutionMetadata.ts`（EvalExecutionContext）未被原包覆盖，Reconciliation 补入 EK 引用（见 reconciliation）
- **Benchmark case**：以 OTel GenAI 适配（KO-02）为 benchmark——独立 Auditor 从零重读 ObservationTypeMapper + OtelIngestionProcessor，可独立重建"注册表+优先级+词表映射"结论，判定 CONFIRMED

## 4. Contradictions & Counterexamples（保留，不删除）
- **无 CONTRADICTED**（原包与独立重建无事实冲突）
- **Counterexample 记录**：
  - KO-01 反例：延迟仅覆盖日期边界，同日乱序不保证——模型范围收窄
  - KO-02 反例：SimpleAttributeMapper 仅单属性键，复合映射是扩展点
  - KO-05 反例：Secondary 语义未证（升级为 C-01）
  - KO-06 反例：minimumReleaseAge 有 Exclude 白名单（next/turbo）——边界开口

## 5. Reconciliation（阶段⑥，合并独立验证修正）
| # | 修正 | 类型 | 原产物变更 |
|---|---|---|---|
| R-1 | EK-28 从"消费隔离/冗余"降为"存在 Secondary 变体，语义未证" | scope 收紧 | ek-graph.md EK-28 表述 + candidates C-01 |
| R-2 | EK-24 从"不依赖自然语言"降为"输出定义编译+校验（仍有插值路径）" | scope 收紧 | ek-graph.md EK-24 |
| R-3 | EK-14 从"dogfooding"降为"内部 trace 写入" | 管道顺序修正 | ek-graph.md EK-14 |
| R-4 | 补 EvalExecutionContext（evalExecutionMetadata.ts）进入 EK-15 证据引用 | 遗漏 | ek-graph.md EK-15 links |
| R-5 | **EK-28 升格实证**：盲重建证明 Secondary 队列 = 按项目 ID 重定向分流（LANGFUSE_SECONDARY_EVAL_EXECUTION_QUEUE_ENABLED_PROJECT_IDS + shardingKey=projectId-jobExecutionId；ingestion 同构） | 实证升格 | ek-graph.md EK-28 更新为已证语义；C-01 关闭 |
| R-6 | **补 EK-45**：eval 执行 LLM 重试预算（retryable → job_executions DELAYED 120min + 确定性 executionTraceId=createW3CTraceId(jobExecutionId)；rate limit → BullMQ exp backoff；非 retryable → ERROR） | 遗漏 | ek-graph.md 新增 EK-45（subsystem→EK-17） |
| R-7 | **补 EK-46**：EvalExecutionContext 元数据键（evaluator_id/evaluator_version_id/evaluator_test/evaluation_rule_id/job_execution_id/target_* 等）——eval 全链路溯源 | 遗漏 | ek-graph.md 新增 EK-46（subsystem→EK-18） |
| R-8 | **补 EK-47**：DecisionModel 答案→Score 严格校验写回（三查：question 存在/type 匹配/choice∈options，任一失败抛 DecisionModelEvaluatorError；CATEGORICAL/NUMERIC 输出） | 遗漏 | ek-graph.md 新增 EK-47（causal→EK-16） |
| R-9 | EK-41 表述收紧："prisma=false（CLI 不 build）但 @prisma/client/@prisma/engines=true"——原写"prisma/esbuild/sharp 等 true"过度概括 | OVER_GENERALIZED 修正 | ek-graph.md EK-41 表述 |
| R-10 | **补 EK-48**：modelMatch 三级链 Local→Redis→Postgres→none + negative token（addModelNotFoundTokenToRedis）——原 EK-05 只写 Local+Redis 失效 | 遗漏 | ek-graph.md 新增 EK-48（mechanism→EK-05） |
| R-11 | C-04 半关闭：概率/置信度下游消费点实证 = 写入 comment（winner/confidence/runner-up 排序）+ score metadata，非独立阈值决策 | 实证 | candidates.md C-04 更新 |
| R-12 | C-05 强化：legacy observation 实体处理在 processEventBatch 未见（grep 无命中），仅 types.ts 定义 + schemaUtils 引用——可能纯类型残留 | 证据强化 | candidates.md C-05 更新 |

## 6. Evidence Register（关键证据索引）
| 机制 | 证据文件 |
|---|---|
| 事件类型 | packages/shared/src/server/ingestion/types.ts:319-337 |
| 摄取延迟 | packages/shared/src/server/ingestion/processEventBatch.ts:59-86 |
| 确定性采样 | packages/shared/src/server/ingestion/sampling.ts:1-50 |
| 模型匹配缓存 | packages/shared/src/server/ingestion/modelMatch.ts:31-58 |
| OTel level 词表 | packages/shared/src/server/otel/OtelIngestionProcessor.ts:29-56 |
| OTel 类型映射 | packages/shared/src/server/otel/ObservationTypeMapper.ts:1-60 |
| OTel 属性 | packages/shared/src/server/otel/attributes.ts:1-50 |
| DecisionModel | packages/shared/src/features/evals/decisionModel.ts:1-80 |
| Eval 治理门 | packages/shared/src/features/evals/evalConfigBlocking.ts:1-90 |
| LLM judge | packages/shared/src/server/evals/llmEvaluatorExecution.ts:1-70 |
| eval 队列 | worker/src/queues/evalQueue.ts:29-345 |
| worker 装配 | worker/src/app.ts:1-60 |
| ClickHouse 事件表 | packages/shared/src/eventsTable.ts:1-47 |
| 供应链治理 | pnpm-workspace.yaml（root） |
| Agent 治理 | AGENTS.md:1-70 |
| Skills 锁定 | skills-lock.json:1-12 |
