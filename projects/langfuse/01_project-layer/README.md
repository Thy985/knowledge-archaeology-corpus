# Project Layer — langfuse

> 项目地图（L0 事实）。全部可回溯至克隆工作区 `repo/` 实际内容。

## 1. 项目定位
- 开源 LLM 工程平台：develop / monitor / evaluate / debug AI applications（README.md）
- 一句话：`Open source agent evals & observability: Trace, evaluate, and improve LLM applications`（GitHub API desc）
- v4.48.0（package.json version）；MIT（package.json license + LICENSE + README badge；GitHub API 标记 NOASSERTION 系检测器差异）
- Y Combinator W23（README badge）

## 2. 架构（monorepo 三进程 + 共享核心）
```
                ┌──────────────────────────────────────────────┐
                │                 web (Next.js)                │
                │  UI + pages/api (public/project/admin/auth)  │
                └──────┬───────────────────────┬───────────────┘
                       │ tRPC/HTTP             │ public API
                ┌──────▼───────────────────────▼───────────────┐
                │        packages/shared (核心)                │
                │  ingestion │ otel │ evals │ features │ prisma│
                │  clickhouse │ redis(queues) │ rbac │ llm     │
                └──────┬───────────────────────┬───────────────┘
                       │ queue jobs            │ queue jobs
                ┌──────▼───────────────────────▼───────────────┐
                │        worker (BullMQ consumers)            │
                │  ingestion/otel/eval/batch/integrations     │
                └──────────────────────────────────────────────┘
  外部数据：Postgres(Prisma 元数据) + ClickHouse(事件) + Redis(队列/缓存) + S3(事件/媒体)
```
- ai-gateway/ 独立包（AI 网关，被 shared 引用其 scope name）
- ee/ 企业版（licenseCheck / ingestionMasking）

## 3. 核心模块
| 模块 | 路径 | 职责 |
|---|---|---|
| Ingestion | packages/shared/src/server/ingestion | 事件校验/批量处理/采样/模型匹配（processEventBatch.ts / sampling.ts / modelMatch.ts） |
| OTel 适配 | packages/shared/src/server/otel | OTel span → 内部 Observation（OtelIngestionProcessor.ts / ObservationTypeMapper.ts / attributes.ts） |
| Eval 定义 | packages/shared/src/features/evals | DecisionModel / blocking / outputDefinition / types |
| Eval 执行 | packages/shared/src/server/evals + worker/src/features/evaluation | LLM judge / DecisionModel 执行 / 代码 eval 双调度器 / evalService |
| 队列 | worker/src/queues + shared server/redis | 25+ BullMQ 队列注册与消费（workerManager.ts / shardedQueueRegistry.ts） |
| 数据模型 | packages/shared/prisma/schema.prisma | 元数据（User→Project→TraceSession/ScoreConfig/Dataset/Prompt/Skill） |
| 事件表 | packages/shared/src/eventsTable.ts | ClickHouse events 表 SQL 构造（trace_name 聚合等） |
| Prompt | packages/shared/src/features/prompts | prompt 名称校验/版本 |
| RBAC | packages/shared/src/features/rbac | projectAccessRights.ts |
| Skills | packages/shared/src/features/skills + packages/langfuse-skills | Langfuse Skills 资产 + skills-lock.json |

## 4. 生命周期（一条 trace 的旅程）
1. **摄取**：SDK/OTel 发送事件 → `web` 或 worker 的 ingestion API → 校验（zod schema）→ 入 Redis ingestionQueue（含延迟策略）
2. **批处理**：worker `processEventBatch` → 采样判定（SHA-256）→ modelMatch（模型/定价）→ 归一化 → ClickHouse events 表写入（+S3 大事件上传）
3. **查询/展示**：web tRPC → ClickHouse/Postgres 查询（tableColumnsToSqlFilter 等过滤构造）
4. **评估**：eval 规则（ACTIVE+非 blocked）→ evalJobCreator → 入队 → evalJobExecutor → LLM-as-judge/DecisionModel/代码 eval → Score 写回
5. **运维**：批量任务（batchExport/dataRetention/blob-cleaner）由独立队列异步执行

## 5. 核心数据结构
- **IngestionEventType**（17 种）：trace/score/event/span/generation/agent/tool/chain/retriever/evaluator/embedding/guardrail create+update、sdk-log、dataset-run-item-create、legacy observation
- **Observation 类型**：SPAN/GENERATION/EVENT/AGENT/TOOL/CHAIN/RETRIEVER/EVALUATOR/EMBEDDING/GUARDRAIL（ObservationTypeDomain）
- **DecisionModel**：CHOICE/SCORE/NOUL 三类问题 + state + questions 字典；limits（maxQuestions=50, maxChoiceOptions=255, maxScoreLevels=10）
- **EvalRuleRunState**：status + blockedAt；EvalExecutionMode = LIVE|MANUAL
- **ModelWithPrices**：Model + PricingTierWithPrices（modelMatch 返回值）

## 6. 测试体系
- 362 个 *.test.ts（vitest）；web 另有 playwright.config.ts（E2E）
- worker 含 integration（productionDeps / awsLambdaCodeEvalDispatcher.integration）
- AGENTS.md 测试纪律：测试匹配"可能无人注意地回归的行为"；bug 先写最小失败测试

## 7. 配置与治理
- **供应链**：pnpm minimumReleaseAge=7200（5 天新依赖延迟）、trustPolicy no-downgrade、allowBuilds 白名单、CVE overrides（undici/hono/@grpc 等）、4 个 patchedDependencies（上游未修）
- **AGENTS.md**：subagent 委派噪音工作、design-system 复用优先、种子 CLI 优先（pnpm run seed）、preview URL、ticket id 不外泄 OSS
- **skills-lock.json**：skill 来源 GitHub + computedHash（哈希锁定）
- **auth**：ApiKey（public API）+ next-auth Session + AuthHeaderValidVerificationResultIngestion
- **SECURITY.md / REVIEW.md**：安全与代码审查声明

## 8. 外部依赖
- Next.js 16 / React 19 / TS 7（+TS6 兼容）/ pnpm 12.6 / turbo 2.11
- BullMQ（Redis）、ClickHouse、Prisma+Postgres、OpenTelemetry、AWS SDK（S3/Lambda）、zod、decimal.js

## 9. 规模
- shared 152,334 行 / web 358,101 行 / worker 150,702 行 TS（合计 ~66 万行，6515 文件）
- stars 35,237；pushed 2026-09-30（昨日活跃）；open issues 984
