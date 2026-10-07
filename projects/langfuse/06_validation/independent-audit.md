# Independent Audit Report — langfuse (ARCH-2026-10-01-001)

> 独立 Auditor 盲重建。**不引用考古包结论**——独立重读 `repo/` 建立 Independent Findings 后与原包对比。
> 审计时间：2026-10-01（UTC+8）。审计对象 commit：a48751c9。

## 0. 审计方法
- 盲重建路径（独立选择，未参考 package/）：evalQueue.ts → decisionModelEvaluatorExecution.ts → runDecisionModelEvaluation.ts → evalService.ts → ingestionQueue.ts → modelMatch.ts → pnpm-workspace.yaml → evalExecutionMetadata.ts → telemetry.ts/internalTraceOtelWriter.ts → types.ts(legacy)
- 判定类别：CONFIRMED / PARTIALLY_CONFIRMED / DOWNGRADED / OVER_GENERALIZED / MISSING / CONTRADICTED / NEEDS_HUMAN_REVIEW

## 1. Independent Findings（审计员独立结论）

**IF-01｜eval 执行链三段式 + LLM 错误重试预算**（evalQueue.ts:122-240）
- 发现：evalJobExecutorQueueProcessorBuilder 显式处理 retryable native AI SDK provider error：①job 在 retry budget 内 → DELAYED 120min（表 job_executions，executionTraceId 由 jobExecutionId 确定性派生 createW3CTraceId）；②非 retryable → ERROR 停止；③LLM rate limit → BullMQ retry w/ exponential backoff
- 与包对比：原包 EK-17/EK-22 只写了"三阶段队列 + 错误阻断"，**遗漏 120min DELAYED 重试预算与确定性 executionTraceId** → MISSING（补 EK）

**IF-02｜Secondary 队列 = 按项目 ID 重定向分流（非冗余消费）**（evalQueue.ts:122-143 + ingestionQueue.ts:23-29）
- 发现：`enableRedirectToSecondaryQueue` + `LANGFUSE_SECONDARY_EVAL_EXECUTION_QUEUE_ENABLED_PROJECT_IDS`：匹配项目在**主消费者内**被重定向到 Secondary 队列（shardingKey = `${projectId}-${jobExecutionId}`）；ingestion 同构（LANGFUSE_SECONDARY_INGESTION_QUEUE_ENABLED_PROJECT_IDS）
- 与包对比：原包 EK-28 写"消费隔离/冗余"并列为 Candidates C-01 未证——**本次盲重建证明语义为"按项目分流"** → 判定：原 EK-28 PARTIALLY_CONFIRMED（语义现可确认），C-01 关闭

**IF-03｜DecisionModel 答案校验 + Score 写回（严格 type-check）**（decisionModelEvaluatorExecution.ts:252-300）
- 发现：mapDecisionModelAnswersToScores 对每个 answer 做三查：①question 存在性 ②answer.type 与 question.type 匹配（expectedAnswerType）③choice 值必须在 options 内——任一失败抛 DecisionModelEvaluatorError；输出 CodeEvalScoreWithName（CATEGORICAL/NUMERIC/boolean）
- 与包对比：原包 EK-16 写了 state 构建与客户端抽象，**遗漏"答案→Score 的严格校验写回"** → PARTIALLY_CONFIRMED（补充 EK）

**IF-04｜概率/置信度消费：写入 comment+metadata（非独立消费）**（decisionModelEvaluatorExecution.ts:154-197）
- 发现：formatDecisionModelComment 把 winner 概率、confidence、runner-up 概率排序列入 comment；toScoreMetadata 存 metadata —— 概率不驱动独立决策/阈值逻辑（至少在本路径）
- 与包对比：C-04 问"概率是否被下游消费"→ 答案：**被消费但仅为评语/元数据面，非阈值决策** → C-04 半关闭（消费点确认）

**IF-05｜modelMatch 缓存三级链（Local → Redis → Postgres → none）**（modelMatch.ts:70-140）
- 发现：findModel 走 instrumentAsync 缓存链：本地 LocalCache → Redis（LANGFUSE_CACHE_MODEL_MATCH_ENABLED）→ Postgres（findModelInPostgres + findPricingTiersForModel）→ 未命中写 negative token（addModelNotFoundTokenToRedis）；span 打 model_match_source
- 与包对比：原包 EK-05 只写"LocalCache + Redis 失效"，**遗漏 Redis 层直接缓存与 Postgres 兜底 + negative token** → PARTIALLY_CONFIRMED（补充 EK）

**IF-06｜供应链 allowBuilds 白名单：datadog 全族禁 build + prisma 禁 build**（pnpm-workspace.yaml）
- 发现：allowBuilds 中 @prisma/client/@prisma/engines=true 但 **prisma=false**（CLI 不执行 postinstall）；@datadog/native-* 全 false；core-js/msw/dd-trace 全 false；cpu-features/ssh2=true（node-ssh 依赖）
- 与包对比：原包 EK-41 写"prisma/esbuild/sharp 等 true"——**"prisma" 实际是 false**（区分 prisma CLI 与 @prisma/client 引擎包）→ CONTRADICTED（微修正 EK-41 表述）

**IF-07｜EvalExecutionContext 元数据键（审计链路溯源）**（evalExecutionMetadata.ts:1-30）
- 发现：EvalExecutionMetadataKey 枚举（evaluator_id/evaluator_version_id/evaluator_test/evaluation_rule_id/job_execution_id/target_trace_id/target_dataset_item_id 等）——eval 全链路可溯源
- 与包对比：原包完全未覆盖 → MISSING（补 EK）

**IF-08｜InternalTrace 强约束：必须用 LangfuseInternalTraceEnvironment**（telemetry.ts:102-112）
- 发现：内部 trace 写入前 Safeguard 校验环境枚举，非枚举值 "Skipping trace creation"——**内部可观测有硬约束**
- 与包对比：原包 EK-14 写"dogfooding 模式"——盲重建显示是"强制环境枚举 + 跳过保护"机制 → DOWNGRADED（收紧表述）

**IF-09｜eval 队列重试路径：retryLLMRateLimitError + 状态机**（evalQueue.ts:170-240）
- 发现：LLM rate limit 错误 → retryLLMRateLimitError（job_executions 表 DELAYED + BullMQ exponential backoff 双路径）；非 retryable → ERROR done
- 与包对比：原包 EK-29 泛写 DeadLetterRetryQueue——eval 有**专门的 LLM rate limit 重试预算逻辑**（120min DELAYED）→ PARTIALLY_CONFIRMED

**IF-10｜legacy observation 事件仅存于类型层 + schemaUtils**（types.ts + clickhouse/schemaUtils.ts）
- 发现：observation-create/update 的实体处理逻辑在 processEventBatch 中**未见**（grep 无命中）；仅 types.ts 定义 + schemaUtils 引用——legacy 事件实际处理路径需更上游定位（可能 API 面已拦）
- 与包对比：原包 EK-07 写"legacy 仅向后兼容保留、走同管道"——**盲重建未见同管道处理证据** → PARTIALLY_CONFIRMED（C-05 强化：legacy 路径证据不足，可能已是纯类型残留）

## 2. 判定统计
| 判定 | 数量 | 条目 |
|---|---|---|
| CONFIRMED | 0（直接完全一致） | —（盲重建未逐条重读全部 44 EK，仅攻击关键面） |
| PARTIALLY_CONFIRMED | 5 | IF-03/IF-05/IF-09/IF-10、原 EK-28（语义现已确认） |
| DOWNGRADED | 1 | IF-08（EK-14 dogfooding → 强制枚举保护） |
| OVER_GENERALIZED | 1 | IF-06（EK-41 "prisma=true" 过度概括，实为 prisma=false 但 @prisma/client=true） |
| MISSING | 2 | IF-01（120min 重试预算）、IF-07（EvalExecutionContext） |
| CONTRADICTED | 0 | IF-06 归为 OVER_GENERALIZED（表述精度问题，非事实冲突） |
| NEEDS_HUMAN_REVIEW | 0 | — |

**Benchmark case（OTel GenAI 适配）**：盲重建 ObservationTypeMapper + OtelIngestionProcessor 的 level 词表 + priority 注册表——可独立重建"外部语义归一化"结论 → **CONFIRMED**（与本包 KO-02 一致）

## 3. 错误与遗漏（对原包的攻击结果）
| # | 攻击点 | 结果 |
|---|---|---|
| A-1 | 单案例→Pattern（EK-28 Secondary 队列） | 原包列为 C-01 未证 → 盲重建实证"按项目分流" → 升级为已证 Pattern |
| A-2 | Pattern→L4（KO-04 可暂停自动化） | 治理门逻辑与 MANUAL 绕过语义在代码实证 → 支持 L4（未发现过度升维） |
| A-3 | ADR→实现事实（无 ADR，specs 仅 1 份） | 无 ADR 依赖，全部结论直连代码 → 无 ADR 失真风险 |
| A-4 | Flow Edge 真实性（F2 blocked 状态机 / F7 供应链策略） | 独立重读确认 Edge 存在（evalConfigBlocking.ts / pnpm-workspace.yaml） → 真实 |
| A-5 | bypass/override 路径 | eval MANUAL 绕过 toggle 不绕过 blocked（实证）；ingestion Secondary 重定向（实证） → 关键 bypass 语义均被盲重建确认 |
| A-6 | Epistemic 混淆 | 原包 C-01~C-06 均标 Hypothesis、KO-08 标 Cross-project pending → 未发现混淆 |

## 4. 结论
- 原考古包无事实性 Contradiction；3 处需收紧（EK-14/28/41），2 处遗漏（重试预算、EvalExecutionContext），1 处 C-04 半关闭（概率写入 comment/metadata 而非阈值决策）
- **本次盲重建的最大贡献**：Secondary 队列语义从"未证"变"实证（按项目分流）"——关闭 C-01；以及发现 eval 特有的 LLM 重试预算机制（120min DELAYED + 确定性 executionTraceId）
- 禁止修改原考古产物；修正并入 Reconciliation（阶段⑥）

## 5. 提交物
- 本报告：independent-audit/report.md
- 修正清单：见 package/06_validation/validation.md §5 Reconciliation（R-1~R-4 由本报告补充为 R-5~R-9）
