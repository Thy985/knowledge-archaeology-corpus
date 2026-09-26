# 05 Candidates — Mem0

> 未确认/跨项目/scope 不确定的内容保留在此，不冒充已验证知识。

## C-01 版本口径差异（v2.1.0 vs "v3.0 算法"）
- **内容**: pyproject version=2.1.0，但代码内有 "V3 PHASED BATCH PIPELINE" 注释与 ADDITIVE_EXTRACTION_PROMPT；候选卡记载"平台 v3.0.0 新算法（单遍 ADD-only 去图存储，09-07 Habr 确认）"；docs/migration/oss-v2-to-v3 存在。
- **状态**: Observation——OSS SDK 版本号与平台 v3 算法命名的关系未在仓库内完全自洽说明；`generate_additive_extraction_prompt` 注释"Ported from platform/backend/shared/core/utils/prompt_builder.py"证明 OSS 与平台代码同源。
- **缺失证据**: v2→v3 迁移文档的具体差异清单（docs/migration/oss-v2-to-v3 未逐条核对）。
- **验证路径**: 读 docs/migration/oss-v2-to-v3 + changelog。

## C-02 notices 系统的真实用途
- **内容**: 七类 notice 当前全部 disabled，但机制完整（远程配置 + A/B variant_split + StaticFlagResult）。推断其用于"OSS→平台转化引导"（first_run/scale_threshold/performance 提示）。
- **状态**: Hypothesis——机制证实，用途是推断。
- **缺失证据**: 无历史 enabled 配置样本；GitHub raw 配置的历史版本不可查（仓库内仅有 bundled 当前版）。
- **验证路径**: 监控 oss_notices_config.json 远程变更 / 检查 docs 或 issue 提及。

## C-03 entity_store 的实现独立性
- **内容**: `_entity_collection_name`（main.py:422）按 provider+collection 派生实体库 collection 名——实体库与记忆库共用同一 vector store provider 但不同 collection。entity boost 检索 top_k=500 的规模成本未量化。
- **状态**: Observation（机制）/ Hypothesis（成本影响）。
- **缺失证据**: 实体库独立 collection 的具体创建路径（未追踪 entity_store 的 init 全链）。
- **验证路径**: 追踪 entity_store 初始化 + 大实体库下的检索延迟。

## C-04 telemetry 迁移信号的业务意图
- **内容**: `_telemetry_vector_store` 用 collection_name="mem0migrations" + migrations_* 路径——遥测记录"迁移"活动。具体捕获了什么事件、如何用于转化漏斗，仓库内无说明。
- **状态**: Observation（实现）/ Hypothesis（商业意图）。
- **缺失证据**: 无 telemetry 事件 schema 文档化迁移信号。

## C-05 md5 hash 去重的理论冲突风险
- **内容**: 去重用 `md5(text)` 精确匹配——理论上不同文本 md5 冲突概率极低但非零；且语义等价但措辞不同的记忆不会被去重（靠 LLM 提示词的 dedup 指引）。
- **状态**: Hypothesis（风险）/ Fact（机制）。
- **缺失证据**: 无冲突处理代码（冲突时静默跳过——潜在漏记忆）；语义去重的实际召回效果无基准。

## C-06 跨项目模式候选（与已有 Corpus 对照）
- **内容**:
  - C-06a: "记忆=压缩事实层" 与 dsh-memory-evolve（本地抽取式）、letta-code（agent 自编辑）的对照——三种记忆写入哲学（自动抽取/主动编辑/文件快照 hermes）。是否构成"外部记忆架构谱系"需跨项目综合。
  - C-06b: KO-04（OSS 漏斗产品化）在 letta/dynamo（同为开源 infra 项目）的对照——OpenAI 系 vs 独立 OSS 的商业化路径差异。
- **状态**: Cross-project Hypothesis——单项目证据不足，需要 2+ 项目联合考古。
- **验证路径**: letta-code 考古结果（PR #18 已合并）对照；dynamo（PR #20）。

## C-07 server 自托管的完整度
- **内容**: server 有 auth/rate_limit/dashboard/telemetry，但 OSS Memory 类与 server 的集成方式（server 是否调用 mem0.memory.Memory？）未深究；docker-compose 引用 pgvector+Neo4j 暗示平台架构（图存储）与 OSS（向量+实体）不同。
- **状态**: Observation——"server 与 OSS SDK 的关系"未闭合。
- **缺失证据**: server main.py 中 Memory 类实例化路径未追踪。
- **验证路径**: server/main.py 全文 + docker-compose 服务定义。
