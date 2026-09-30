# Engineering Knowledge Graph — langfuse

> EK Graph（v3.1 契约）：每条 EK 声明 links（6 类边），KO 从图上按 R1-R4 聚合。证据格式 `file:行号` 或 `file`（文件级，可回溯至 `repo/` 工作区）。
> 层级：L1（工程事实）/ L2（工程知识）。
> **v2（Reconciliation 后）**：新增 EK-45~48（独立 Auditor 盲重建补充），EK-28/EK-41 表述修正。

## 图概览
- EK 总数：48（EK-01 ~ EK-48）
- links 声明率：100%（48/48）；游离 EK：0
- 子系统：A=Ingestion（8）、B=OTel 适配（6）、C=Eval（14）、D=Worker 架构（6）、E=数据与查询（5）、F=供应链与治理（4）、G=Agent 协同（3）、H=补充（4）

---

## 子系统 A — Ingestion（摄取管道）

**EK-01｜事件类型系统（17 种事件）｜L1**
事件被规格化为带类型的事件流：trace-create/score-create/event-create/span-create|update/generation-create|update/agent-create/tool-create/chain-create/retriever-create/evaluator-create/embedding-create/guardrail-create/sdk-log/dataset-run-item-create + legacy observation-create|update。
证据：packages/shared/src/server/ingestion/types.ts:319-337（eventTypes 定义，含 LEGACY 注释"only required for backwards compatibility"）
links: mechanism→EK-02（同是事件化摄取的核心语义单元）；subsystem→EK-02/EK-03/EK-04/EK-05

**EK-02｜processEventBatch 批量摄取核心｜L2**
事件批次在 worker 侧统一处理：校验（zod）→ 采样判定（EK-06）→ 模型匹配（EK-07）→ 归一化 → 写入 ClickHouse；支持 S3 事件上传（大事件溢出路径，LANGFUSE_S3_EVENT_UPLOAD_* 配置）与 s3SlowdownTracking（S3 慢速标记 + markProjectIngestFailure 失败计数）。
证据：packages/shared/src/server/ingestion/processEventBatch.ts:1-100（imports 含 s3SlowdownTracking/ingestionFailureTracking/StorageService；getS3StorageServiceClient 惰性单例）
links: dependency→EK-06（采样判定依赖）；dependency→EK-07（模型匹配依赖）；mechanism→EK-09（都是"批次输入→归一化→落库"管道）

**EK-03｜摄取延迟策略（防跨日乱序）｜L2**
getDelay：delay 显式覆盖优先；UTC 23:45~00:15 之间强制使用 env delay（跨日边界防重复/乱序）；OTel 源 0 延迟；API 源 min(5000, env delay)（避免 worker 重复处理）。
证据：packages/shared/src/server/ingestion/processEventBatch.ts:59-86（getDelay 注释 "avoid duplicates for out-of-order processing of events"）
links: causal→EK-02（延迟决定入队时机）；constraint→EK-08（时间边界约束与事件时间语义相关）

**EK-04｜S3 事件上传溢出路径｜L2**
大/特殊事件可经 S3 上传（StorageService 惰性单例、forcePathStyle/SSE 配置），避免事件体超出队列载荷——摄取管道对"体积边界"提供显式溢出通道。
证据：packages/shared/src/server/ingestion/processEventBatch.ts:38-57（getS3StorageServiceClient）
links: contrast→EK-02（同管道内"直写 vs S3 溢出"两种路径）

**EK-05｜模型匹配缓存链（modelMatch）｜L2**
模型→价格解析带两级缓存：本地 LocalCache（TTL-only，默认 10s/20000 条，跨容器一致性靠 Redis 失效 + 短 TTL）+ Redis 失效信号（LOCK:model-match-clear）；缓存命中避免每事件查库。
证据：packages/shared/src/server/ingestion/modelMatch.ts:31-58（"This L1 cache is intentionally TTL-only. Cross-container consistency continues to come from Redis invalidation plus the short local TTL."）
links: subsystem→EK-02；constraint→EK-44（缓存一致性约束与供应链"降级"语义对照）

**EK-06｜确定性 SHA-256 采样｜L2**
采样按 traceId 确定性判定：SHA-256(traceId) 前 8 字节 → 归一化 [0,1) → 与采样率比较；未配置项目/无效采样率（<0 或 >1）保守保留（isSampled=true）；LANGFUSE_INGESTION_PROCESSING_SAMPLED_PROJECTS 按项目配置。
证据：packages/shared/src/server/ingestion/sampling.ts:1-50（isInSample：hashInt/0xffffffff < sampleRate；"Be conservative and keep the trace ID in sample for invalid configs"）
links: dependency→EK-02（批处理调用采样）；mechanism→EK-10（都是"确定性函数 + 保守降级"）

**EK-07｜事件 schema 双轨（新事件 + LEGACY 兼容）｜L2**
现代事件类型（trace/span/generation/agent/tool...）与 legacy observation-create/update 并存；legacy 仅向后兼容保留——新老 API 面共享同一批处理管道。
证据：packages/shared/src/server/ingestion/types.ts:337-340（"// LEGACY, only required for backwards compatibility" + LegacySpanPostSchema）
links: contrast→EK-01（同一系统的两代事件语义）；constraint→EK-02（legacy 事件也走同管道）

**EK-08｜事件时间语义与跨日边界｜L2**
摄取延迟与事件时间戳围绕 UTC 日期边界设计（23:45-00:15 特殊处理），保证 out-of-order 事件在日期切换时不被重复计数/丢失。
证据：processEventBatch.ts:59-86（getDelay 跨日注释）
links: causal→EK-03；subsystem→EK-02

---

## 子系统 B — OTel 适配

**EK-09｜OTel GenAI 摄取处理器（OtelIngestionProcessor）｜L2**
外部 OTel span（含 GenAI semconv）经处理器归一化为内部事件流；带内部状态 TraceState（hasFullTrace / shallowEventIds）跟踪"是否已见完整 trace"。
证据：packages/shared/src/server/otel/OtelIngestionProcessor.ts:57-80（TraceState 接口 + AI_GATEWAY_INSTRUMENTATION_SCOPE_NAME = "langfuse-ai-gateway"）
links: mechanism→EK-02（同"外部输入→归一化→管道"）；subsystem→EK-10/EK-11/EK-12

**EK-10｜外部 level 词表映射（保守降级）｜L2**
OTel 外来 level 词汇（DEBUG/TRACE/VERBOSE/DEFAULT/INFO/LOG/NOTICE/OK/SUCCESS/WARNING/WARN/ERROR/FATAL/CRITICAL——覆盖 OTel severity、python logging、loguru、console）映射到 Langfuse 内部 ObservationLevel 枚举；未知值返回 undefined 由调用方 status 兜底（不掩盖成 DEFAULT）。
证据：packages/shared/src/server/otel/OtelIngestionProcessor.ts:29-56（OBSERVATION_LEVEL_ALIASES + 注释 "Unknown values return undefined so the caller's status-derived fallback applies instead of masking it with DEFAULT"）
links: contrast→EK-11（词表映射 vs 类型映射，两种适配）；mechanism→EK-15（都对外部语义做保守归一化）

**EK-11｜ObservationTypeMapper 注册表（priority 排序）｜L2**
OTel 属性→内部 Observation 类型通过注册表（带 priority，低值优先）逐 mapper 尝试：SimpleAttributeMapper（单属性键→类型映射，经 ObservationTypeDomain.safeParse 校验）；canMap 先行，mapToObservationType 后行。
证据：packages/shared/src/server/otel/ObservationTypeMapper.ts:1-60（interface 含 priority；SimpleAttributeMapper.canMap 检查属性存在且有意义值）
links: mechanism→EK-09（同适配层）；contrast→EK-10

**EK-12｜Langfuse OTel span 属性枚举｜L1**
内部约定的 span 属性键（langfuse.* 前缀 + user.id/session.id + 兼容属性 TRACE_COMPAT_USER_ID=langfuse.user.id 等）作为 OTel 摄入与写出的契约面。
证据：packages/shared/src/server/otel/attributes.ts:1-50（LangfuseOtelSpanAttributes 枚举：TRACE_NAME/OBSERVATION_TYPE/EXPERIMENT_ID 等 + Compatibility 注释）
links: subsystem→EK-09/EK-11；constraint→EK-10（属性面约束 level 解析）

**EK-13｜OTel 摄取 worker 特征（processOtelEvents）｜L2**
worker 侧独立 otel-ingestion feature：OTel 事件批量处理（otelsev 独立队列），与 API ingestion 分流但共用归一化产物。
证据：worker/src/features/otel-ingestion/processOtelEvents.ts + README.md；worker/src/app.ts（OtelIngestionQueue/SecondaryOtelIngestionQueue 注册）
links: contrast→EK-02（OTel vs API 两套摄取入口）；subsystem→EK-09

**EK-14｜InternalTrace 写入（langfuse 内部 trace 化）｜L2**
平台自身操作（如 prompt experiments 产生的内部事件）被写入内部 trace（internalTraceOtelWriter/internalAiFeatureOtelWriter），即"平台自身可观测"——dogfooding 模式。
证据：packages/shared/src/server/otel/internalTraceOtelWriter.ts + internalAiFeatureOtelWriter.ts（存在 + 各自测试）
links: mechanism→EK-09（复用 OTel 写入面）；contrast→EK-43（自观测 vs 外部观测）

---

## 子系统 C — Eval

**EK-15｜DecisionModel：结构化问题规格｜L2**
eval 被定义为结构化问题集而非自由文本：CHOICE（选项+概率）/ SCORE（分级）/ NOUL（boolean+概率）三类；state 变量（STATE_KEY_PATTERN 正则校验）+ questions 字典；limits 硬上限（50 问题/255 选项/10 分级/20000 指令长度）。
证据：packages/shared/src/features/evals/decisionModel.ts:1-80（DecisionModelQuestionType/limits/STATE_KEY_PATTERN/ChoiceQuestionSchema refine 唯一选项）
links: causal→EK-16（问题规格驱动执行）；mechanism→EK-21（同"规格化输出定义"思想）

**EK-16｜DecisionModel 执行（state 构建 + 客户端抽象）｜L2**
执行端：ExtractedVariable → buildDecisionModelState（null/undefined 跳过）→ DecisionModelRequest{state, questions} → DecisionModelClient.evaluate（抽象客户端，返回 answers{type+probabilities+confidence} + usage）——judge 输出带概率与置信度。
证据：packages/shared/src/server/evals/decisionModelEvaluatorExecution.ts:1-90（DecisionModelRequest/Answer 类型 + buildDecisionModelState "if (variable.value === null ...) continue"）
links: causal→EK-15；dependency→EK-17（执行结果供 evalScoreEvent 写回）

**EK-17｜eval 执行三阶段队列（Trace/Dataset Creator→Executor→LLM judge）｜L2**
eval 管道分队列阶段：evalJobTraceCreatorQueueProcessor（trace 目标）→ evalJobCreatorQueueProcessor（配置展开）→ evalJobExecutorQueueProcessorBuilder（执行）→ LLMAsJudgeExecutionQueueProcessor（judge 调用）——每阶段独立队列可独立重试/扩缩。
证据：worker/src/queues/evalQueue.ts:29-345（四个 processor 导出 + jobName EvaluationExecution/LLMAsJudgeExecution）
links: causal→EK-18（执行产出 Score）；subsystem→EK-19/EK-20

**EK-18｜evalService：规则→执行 编排｜L2**
worker evalService 负责规则评估、变量提取、目标解析（trace/dataset 双目标）、eval 数据构建；含 requiresDatabaseLookup/inMemoryFilterRequiresMetadata 判定（过滤是否需要查库）——内存过滤优先，减少 DB 往返。
证据：worker/src/features/evaluation/evalService.ts:1-60（imports 含 requiresDatabaseLookup/inMemoryFilterRequiresMetadata/checkTraceExistsAndGetTimestamp/blockEvaluator）
links: causal→EK-17；dependency→EK-16

**EK-19｜EvalRule 治理门（LIVE/MANUAL + blocked）｜L2**
eval 规则运行状态 = status(ACTIVE) + blockedAt；isEvalRuleExecutable = ACTIVE && !blocked；canRunEvalRule：MANUAL 模式绕过 live-traffic toggle 但**不**绕过 blocked 规则（"Manual batch runs bypass only the live-traffic toggle, not blocked rules"）——治理分层显式化。
证据：packages/shared/src/features/evals/evalConfigBlocking.ts:1-50（EvalExecutionMode z.enum(["LIVE","MANUAL"]) + canRunEvalRule 注释）
links: causal→EK-18（门控决定执行与否）；constraint→EK-20（block 元数据约束用户修复路径）

**EK-20｜EvaluatorBlock 原因分类与用户可读元数据｜L2**
blockedAt + EvaluatorBlockReason（LLM_CONNECTION_AUTH_INVALID 等）→ EVALUATOR_BLOCK_METADATA（message + shortLabel，用户可读修复指引："Update the LLM connection used by this evaluator and then reactivate it"）。
证据：packages/shared/src/features/evals/evalConfigBlocking.ts:55-90（EVALUATOR_BLOCK_METADATA Record）
links: constraint→EK-19；causal→EK-22（block 分类驱动 LLM 错误分类）

**EK-21｜LLM-as-judge：prompt 模板编译 + 输出定义校验｜L2**
executeLlmEvaluator：PersistedEvaluatorPromptMessages（system/user/assistant 角色）→ CHAT_MESSAGE_BUILDERS → compileEvalPrompt 变量插值 → compilePersistedEvalOutputDefinition + validateEvalOutputResult（judge 输出结构校验，非仅自由文本）。
证据：packages/shared/src/server/evals/llmEvaluatorExecution.ts:1-70（CHAT_MESSAGE_BUILDERS satisfies Record；buildEvalMessages/interpolatedPrompt）
links: mechanism→EK-15（同"结构化解耦 judge 输出"）；causal→EK-18

**EK-22｜LLM 错误分类与评估器阻断联动｜L2**
classifyEvaluatorLlmError（LLM 调用失败分类）+ blockEvaluator：judge 调用持续失败（如认证无效）→ 自动阻断评估器并暴露分类原因。
证据：packages/shared/src/server/evals/classifyEvaluatorLlmError.ts（存在）+ evalService.ts imports（classifyEvaluatorLlmError/blockEvaluator/EvaluatorBlockSource）
links: causal→EK-20；dependency→EK-17

**EK-23｜代码 eval 双调度器（local / AWS Lambda）｜L2**
代码型评估经 codeEvalDispatchers 抽象分派：localCodeEvalDispatcher（本地执行）vs awsLambdaCodeEvalDispatcher（远程 Lambda）——同一接口两实现（codeEvalDispatcherTypes 定义契约）。
证据：packages/shared/src/server/evals/codeEvalDispatcherTypes.ts + localCodeEvalDispatcher.ts + awsLambdaCodeEvalDispatcher.ts（存在 + worker codeBased/ 目录）
links: contrast→EK-21（代码 eval vs LLM judge 两执行形态）；mechanism→EK-24

**EK-24｜eval 输出定义编译（CompiledEvalOutputDefinition）｜L2**
PersistedEvalOutputDefinition → compilePersistedEvalOutputDefinition 编译 → CompiledEvalOutputDefinition 供 callLlm 结构化消费——eval 输出有 schema（不依赖自然语言解析）。
证据：packages/shared/src/features/evals/outputDefinition.ts（存在 + outputDefinition.test.ts）+ llmEvaluatorExecution.ts import
links: mechanism→EK-15/EK-21；dependency→EK-21

**EK-25｜eval 实验/数据集变量映射｜L2**
availableTraceEvalVariables / availableDatasetEvalVariables / evalDatasetFormFilterCols / variableMapping：trace 与 dataset 两类 eval 目标的变量面与过滤列显式声明。
证据：worker/src/features/evaluation/evalService.ts imports（Prisma.singleFilterList/variableMappingList/availableDatasetEvalVariables）
links: subsystem→EK-18/EK-17

**EK-26｜eval 阻塞展示状态（PausedEvaluatorDisplayState）｜L2**
显示层引入 PAUSED 作为 JobConfigState 之外的展示状态；EvaluatorExecutionStatusCount 按 evaluator 聚合执行状态计数（UI 审计）。
证据：packages/shared/src/features/evals/evalConfigBlocking.ts:6-30（PausedEvaluatorDisplayState + EvaluatorExecutionCountsByEvaluatorId）
links: subsystem→EK-19/EK-20

---

## 子系统 D — Worker 架构

**EK-27｜25+ BullMQ 队列的 worker 装配（WorkerManager）｜L2**
worker/src/app.ts 集中注册全部队列处理器：ingestion/otel-ingestion/eval（creator/executor/judge）/batchExport/dataRetention/blob-cleaner/experiment/trace-delete/in-app-agent/monitor/cloud 计量等；WorkerManager + drainAndClose/onShutdown 管理生命周期。
证据：worker/src/app.ts:1-60（imports 全部 queue processor + WorkerManager；worker/src/queues/workerManager.ts 存在）
links: mechanism→EK-17（都走队列）；subsystem→EK-28/EK-29

**EK-28｜Secondary 队列（按项目 ID 重定向分流）｜L2**
Secondary 队列语义（Reconciliation 实证）：`enableRedirectToSecondaryQueue` + `LANGFUSE_SECONDARY_{EVAL_EXECUTION,INGESTION}_QUEUE_ENABLED_PROJECT_IDS`——匹配项目在**主消费者内**被重定向到 Secondary 队列（shardingKey = `${projectId}-${jobExecutionId}`）；即按项目分流隔离，非冗余消费/负载均衡。
证据：worker/src/queues/evalQueue.ts:122-143（evalJobExecutorQueueProcessorBuilder enableRedirectToSecondaryQueue + "Redirecting evaluation execution job to secondary queue for project"）+ worker/src/queues/ingestionQueue.ts:23-29（同构）
links: constraint→EK-27（装配约束）；contrast→EK-04（溢出/分流两形态）

**EK-29｜DeadLetterRetryQueue（死信重试）｜L2**
失败任务进入 DeadLetterRetryQueue（DLQ 重试路径），配合队列级重试——"失败不静默丢"的管道保证。
证据：worker/src/app.ts imports（DeadLetterRetryQueue）
links: causal→EK-27（失败路径）；mechanism→EK-30

**EK-30｜Ingestion 失败跟踪（markProjectIngestFailure）｜L2**
摄取失败按项目标记（ingestionFailureTracking Redis）——失败可见、可治理（与 S3 慢速标记同理）。
证据：packages/shared/src/server/ingestion/processEventBatch.ts:24-25（import markProjectIngestFailure from "../redis/ingestionFailureTracking"）
links: subsystem→EK-02/EK-29

**EK-31｜集成 egress 队列（analyticsIntegrationEgress）｜L2**
分析集成（mixpanel/posthog/analytics-integrations）经独立 egress feature 异步外发（outbound-url allowlist + log redaction 测试覆盖）。
证据：worker/src/features/analyticsIntegrationEgress.ts + __tests__（analyticsIntegrationOutboundUrlAllowlist/LogRedaction 测试）
links: contrast→EK-27（内部处理 vs 外部 egress 分型）；subsystem→EK-27

**EK-32｜批量运维队列族（batch-*）｜L2**
batchExport/batch-action/batch-project-cleaner/batch-project-blob-cleaner/batch-trace-deletion-cleaner/media-retention-cleaner——运维/清理类工作全部队列化异步执行，与请求路径完全解耦。
证据：worker/src/features/（batchExport/batchAction/batch-project-cleaner 等目录）+ queues（batchExportQueue/batchActionQueue）
links: subsystem→EK-27；mechanism→EK-31

---

## 子系统 E — 数据与查询

**EK-33｜ClickHouse events 表 + SQL 构造层｜L2**
事件存 ClickHouse（eventsTable.ts 构造 SQL：parent_span_id 判根、trace_name 聚合 argMaxIf、空串↔null 归一化 normalizeEventsTraceName）；tableColumnsToSqlFilterAndPrefix 过滤列→SQL 映射。
证据：packages/shared/src/eventsTable.ts:1-40（eventsTableTraceNameAggregationSqlForAlias argMaxIf 等 + LFE-14924 AMBIGUOUS_COLUMN_NAME 教训注释）
links: dependency→EK-02（批处理落库目标）；causal→EK-34

**EK-34｜wire 空串 ↔ null 归一化（格式一致性）｜L2**
eventsTableTraceNameSelectSql 在 wire 层用 '' 表示无 trace name（ClickHouse 类型一致防 AMBIGUOUS_COLUMN_NAME）；JS 消费面 normalizeEventsTraceName 把 '' 归一为 null；Blob 导出**刻意不**归一化（其契约类型是 plain string，三格式必须一致）。
证据：packages/shared/src/eventsTable.ts:20-47（"Blob storage export deliberately does NOT use this ... would make the three export formats disagree"）
links: constraint→EK-33；contrast→EK-39

**EK-35｜Prisma 元数据模型（组织/项目/密钥/会话）｜L2**
Postgres 元数据：Account/Session/User/VerificationToken/Organization/Project/ApiKey + TraceSession + ScoreConfig + AnnotationQueue + CronJobs + Prompt(+Dependency/ProtectedLabels) + Skill(+File/Blob) + Model/Price/PricingTier；Project 关联 TraceSession/ScoreConfig/Dataset 等。
证据：packages/shared/prisma/schema.prisma（model 列表 + InAppAgentPendingToolApproval 完整定义）
links: dependency→EK-02（摄取落库参照元数据）；subsystem→EK-36/EK-38

**EK-36｜RBAC：projectAccessRights｜L2**
项目级访问权限显式声明（projectAccessRights.ts）——组织/项目成员两级权限面。
证据：packages/shared/src/features/rbac/projectAccessRights.ts（存在）
links: subsystem→EK-35；constraint→EK-37

**EK-37｜认证三通道（ApiKey / next-auth / AuthHeader 校验）｜L2**
public API 走 ApiKey；web 走 next-auth Session（含 nodemailer 自定义 sendVerificationRequest 等定制）；摄取认证用 AuthHeaderValidVerificationResultIngestion 类型化结果。
证据：packages/shared/src/server/auth/types.ts（AuthHeaderValidVerificationResultIngestion）+ prisma ApiKey/Session/VerificationToken + pnpm-workspace overrides（next-auth>nodemailer 注释）
links: subsystem→EK-36；constraint→EK-02（摄取需认证）

**EK-38｜Prompt 管理（名称 pipe 限制 + 保留名）｜L2**
Prompt 名称校验：pipe 字符被禁止（"pipe character is used for prompt composition"——组合语法保留）；RESERVED_PROMPT_NAMES 保留 /prompts/* 页面名；文件夹路径验证 withFolderPathValidation。
证据：packages/shared/src/features/prompts/validation.ts:1-30（PromptNameSchema + 注释）+ constants.ts
links: subsystem→EK-35；contrast→EK-15

**EK-39｜本地过滤优先 + DB 兜底（requiresDatabaseLookup）｜L2**
eval 过滤：inMemoryFilterRequiresMetadata（需要元数据→内存过滤）vs requiresDatabaseLookup（否则查库）——"能内存就不查库"的性能策略。
证据：worker/src/features/evaluation/evalService.ts imports + traceFilterUtils.ts（存在）
links: contrast→EK-34；dependency→EK-18

---

## 子系统 F — 供应链与治理

**EK-40｜pnpm 5 天新依赖延迟 + no-downgrade｜L2**
pnpm-workspace：minimumReleaseAge=7200s（"5 day delay for new dep upgrades to reduce supply chain attack risk"）；trustPolicy no-downgrade（拒绝比已装版本信任弱的近期发布）；minimumReleaseAgeExclude 白名单（next/turbo 等工具链）。
证据：pnpm-workspace.yaml:1-30（注释 + trustPolicyIgnoreAfter）
links: mechanism→EK-41；constraint→EK-42

**EK-41｜allowBuilds 白名单（依赖构建治理）｜L2**
只允许必要依赖执行 postinstall 构建：@prisma/client/@prisma/engines=true 但 **prisma=false**（CLI 不执行构建，Reconciliation 实证）；esbuild/sharp/cpu-features/ssh2/vue-demi=true；@datadog/native-* 全族 false、core-js/msw/dd-trace/protobufjs/@scarf/scarf=false——减少供应链执行面。
证据：pnpm-workspace.yaml（allowBuilds 块，含 dd-trace/msgpackr-extract 注释）
links: mechanism→EK-40；subsystem→EK-42

**EK-42｜CVE overrides + 上游未修 patchedDependencies｜L2**
大量 overrides 钉死 CVE 修复版本（undici/hono/@grpc/grpc-js/ip-address/path-to-regexp/fflate/fast-uri 等，每条带 CVE 号注释）；4 个 patchedDependencies（@radix-ui react-roving-focus/react-select、next-auth、react-resizable-panels——上游未修，本地打丁并注明"remove when we bump"）。
证据：pnpm-workspace.yaml（overrides 块每条 CVE 注释 + patchedDependencies 块 4 条）
links: causal→EK-40（延迟策略的落地执行）；constraint→EK-44

**EK-43｜skills-lock.json 哈希锁定｜L2**
外部 skill（GitHub 来源）以 computedHash 锁定（source/sourceType/skillPath/computedHash 结构化）——skill 供应链溯源。
证据：skills-lock.json:1-12（grill-me 条目）
links: contrast→EK-41（依赖 vs skill 两类供应链治理）；subsystem→EK-40

**EK-44｜AGENTS.md Agent 工作准则（委派/测试/种子/不外泄）｜L2**
仓库自带 Agent 治理规范：subagent 委派噪音工作（"Delegate exploratory or noisy work ... to a subagent"）；测试匹配风险而非变更事实（"A test earns its place when it pins behavior that could regress without anyone noticing"）；bug 先最小失败测试；种子 CLI 优先（pnpm run seed，禁止 ad-hoc 脚本/裸 ClickHouse insert）；ticket id 不外泄 OSS；bot review 不回复直到修复。
证据：AGENTS.md:1-70（Scope/How To Work 全文）
links: mechanism→EK-40（同为治理机制）；constraint→EK-27（治理约束开发流程）

---

## 图统计
- 平均出边：48 条 EK 共 78 条 links，平均 1.63 边/EK
- 游离 EK：0
- 六类边覆盖：mechanism(15) / subsystem(23) / causal(10) / dependency(9) / constraint(9) / contrast(10)

---

## 子系统 H — 补充（Reconciliation 新增，Independent Auditor 实证）

**EK-45｜eval 执行 LLM 重试预算（120min DELAYED + 确定性 executionTraceId）｜L2**
eval 执行失败按错误分类走重试预算：retryable native AI SDK provider error → job_executions 状态 DELAYED + 120min 延迟重试，executionTraceId 由 jobExecutionId 确定性派生（createW3CTraceId）；LLM rate limit → retryLLMRateLimitError（BullMQ exponential backoff）；非 retryable → ERROR 停止。
证据：worker/src/queues/evalQueue.ts:170-240（注释流程图 "DELAYED Retry by 120 min" + executionTraceId = createW3CTraceId(job.data.payload.jobExecutionId)）
links: subsystem→EK-17（eval 队列族）；dependency→EK-22（错误分类驱动重试预算）

**EK-46｜EvalExecutionContext 元数据键（eval 全链路溯源）｜L2**
eval 执行上下文元数据键枚举：evaluator_id/evaluator_version_id/evaluator_test/evaluation_rule_id/evaluation_rule_assignment_id/job_execution_id/job_configuration_id/target_trace_id/target_observation_id/target_dataset_item_id——每条 eval 执行可溯源到规则/版本/目标。
证据：packages/shared/src/features/evals/evalExecutionMetadata.ts:1-30（EvalExecutionMetadataKey 枚举 + EvalExecutionContext 类型）
links: subsystem→EK-18（evalService 消费）；constraint→EK-21（judge 执行携带上下文）

**EK-47｜DecisionModel 答案→Score 严格校验写回｜L2**
mapDecisionModelAnswersToScores 对每个 answer 三查：①question 存在 ②answer.type 与 question.type 匹配（expectedAnswerType）③choice 值∈options；任一失败抛 DecisionModelEvaluatorError；输出 CodeEvalScoreWithName（CATEGORICAL/NUMERIC/boolean）。
证据：packages/shared/src/server/evals/decisionModelEvaluatorExecution.ts:252-300（三查 + "Decision model returned ... which is not an option of question"）
links: causal→EK-16（执行结果→Score）；mechanism→EK-21（结构化输出校验同思想）

**EK-48｜modelMatch 三级缓存链 + negative token｜L2**
模型匹配完整链：LocalCache → Redis（LANGFUSE_CACHE_MODEL_MATCH_ENABLED）→ Postgres（findModelInPostgres + findPricingTiersForModel）→ 未命中写 negative token（addModelNotFoundTokenToRedis）；span 打 model_match_source 审计。
证据：packages/shared/src/server/ingestion/modelMatch.ts:70-140（source: redis/postgres/none + addModelNotFoundTokenToRedis）
links: mechanism→EK-05（缓存链延伸）；constraint→EK-33（落库前定价解析）
