# Candidates — langfuse（未验证假说）

> 保持 Hypothesis 标记。不与 KO 混淆。每条含：假设 / 当前证据 / 缺失证据 / 验证路径。
> **Reconciliation 更新**：C-01 已实证关闭；C-04/C-05 证据强化。

## C-01｜Secondary 队列语义 —— **已实证关闭（Reconciliation R-5）**
- ~~假设：Secondary* 队列用于消费隔离或冗余消费者~~
- **实证结果**：独立 Auditor 盲重建（evalQueue.ts:122-143 + ingestionQueue.ts:23-29）证明 = **按项目 ID 重定向分流**：`LANGFUSE_SECONDARY_{EVAL_EXECUTION,INGESTION}_QUEUE_ENABLED_PROJECT_IDS` 匹配项目在主消费者内被重定向到 Secondary 队列，shardingKey=`${projectId}-${jobExecutionId}`。已并入 EK-28。

## C-02｜"可暂停的自动化"跨项目对照（webwright 完成门 / guardian 策略门 / agent-framework）
- **假设**：langfuse 的 LIVE/MANUAL+blocked 与 webwright 的完成授权门、guardian 策略门、agent-framework 的安全执行模式构成同一治理模式族——"自动化暂停是第一类对象"。
- **当前证据**：langfuse evalConfigBlocking.ts:41-50（本仓）；webwright/guardian/agent-framework 考古结论（corpus 已有）。
- **缺失证据**：三项目治理门的具体语义差异未系统对比（trigger 类型/绕过语义/恢复路径）。
- **验证路径**：跨 corpus 04_flow-atlas 对读 F7 Policy Flow。

## C-03｜双存储分层（Postgres 元数据 + ClickHouse 事件）跨项目同构
- **假设**：与 deepseek-harness 的事件存储、webwright 的 run-artifact 存储同属"元数据/事件分离"模式。
- **当前证据**：本仓 prisma schema（35+ 元数据模型）+ eventsTable.ts（ClickHouse 事件）+ modelMatch 缓存链。
- **缺失证据**：deepseek-harness/webwright 存储层细节需重读对照。
- **验证路径**：corpus 三项目 01_project-layer 对读。

## C-04｜DecisionModel 概率输出下游消费 —— **半关闭（Reconciliation R-11）**
- **假设**：judge 返回的概率/置信度被用于 eval 结果置信度展示或阈值决策。
- **实证结果**：盲重建（decisionModelEvaluatorExecution.ts:154-197, 252-300）证明概率/置信度被消费但**仅为评语+元数据面**：formatDecisionModelComment 将 winner 概率、confidence、runner-up 概率排序写入 comment；toScoreMetadata 存 metadata。**未发现独立阈值决策消费点**（此路径）。
- **剩余未验证**：web UI 是否将 comment/metadata 中的概率渲染为置信度展示（未追踪 web 消费面）。

## C-05｜legacy observation 事件的双轨成本 —— **证据强化（Reconciliation R-12）**
- **假设**：legacy observation-create/update 与新事件类型并存带来持续的 schema 维护成本。
- **当前证据**：types.ts:337-340 LEGACY 注释 + LegacySpanPostSchema；**盲重建新增**：processEventBatch 中 grep 无 legacy observation 实体处理命中，仅 types.ts 定义 + clickhouse/schemaUtils.ts 引用——legacy 路径可能已是纯类型残留（API 面可能已拦截）。
- **缺失证据**：无历史 commit 佐证（浅克隆 depth=1），无法量化维护成本，也无法确认 API 面是否仍接受 legacy 事件。
- **验证路径**：读完整迁移历史（需 full clone）+ web/src/pages/api/public 的 observation 端点。

## C-06｜ee/licenseCheck 与企业版 gating 的边界
- **假设**：企业版特性（ingestionMasking/licenseCheck）通过 license 校验 gating，OSS 与 EE 共享代码面。
- **当前证据**：packages/shared/src/server/ee/ 仅 licenseCheck/index.ts + ingestionMasking 两文件。
- **缺失证据**：licenseCheck 的校验逻辑与 gating 位置未读（EE 目录刻意未深挖——考古范围声明）。
- **验证路径**：按需读 ee/licenseCheck/index.ts + web 端 entitlement（plans.ts）。
