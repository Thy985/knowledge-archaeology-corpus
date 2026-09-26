# 00 Overview — Mem0 (ARCH-2026-09-21-001)

## 一句话定位
Mem0（"mem-zero"）是 AI agent 的长期记忆基础设施——托管平台 API（`api.mem0.ai`）+ 自托管开源 SDK（`mem0ai`）双形态；OSS 侧核心是"LLM 抽取式记忆"：对话 → 单遍 ADD-only 抽取 → 向量库 + SQLite 历史 + 实体库三库落盘 → 混合检索召回。

## 核心命题
**"agent 记忆如何从'全量对话记录'变成'可检索、可更新、可去重的事实层'？"**
Mem0 的回答：不存原始对话（仅存最近 10 条作上下文），而是每次用 LLM 把对话**压成自包含事实语句**（ADD-only），用 hash 去重，用混合检索（语义+BM25+实体 boost）召回——**记忆是压缩后的结构，不是原始回放**。

## 仓库事实
| 项 | 值 |
|---|---|
| repository | github.com/mem0ai/mem0 |
| commit | `a39a802bbc93e85b820078cd3c4dbaf53af25dbe`（2026-09-18） |
| version | mem0ai 2.1.0（pyproject.toml） |
| license | Apache-2.0 |
| 规模 | 57MB / 1812 files / mem0 包 31,848 行 Python / 148 py files |
| 形态 | 核心 SDK（mem0/）+ TS SDK（mem0-ts/）+ CLI×2 + 自托管 server/（FastAPI）+ 16 集成 + skills/ |
| skill version | knowledge-archaeology v3.2 |

## Top 5 发现
1. **V3 单遍 ADD-only 抽取管道**（`_add_to_vector_store` 内 "V3 PHASED BATCH PIPELINE"）：Phase 0 上下文 → Phase 1 现有记忆检索（UUID→整数映射防幻觉）→ Phase 2 单次 LLM 抽取（ADDITIVE_EXTRACTION_PROMPT）→ Phase 3 批量 embed → Phase 4-5 hash 去重 → Phase 6 批量落库。**一次 LLM 调用完成抽取+决策**。
2. **混合检索三信号**（scoring.py `score_and_rank`）：语义（over-fetch 4x）+ BM25（查询长度自适应 sigmoid 归一化）+ 实体 boost（spaCy 提取 → 实体库 0.5 阈值检索 → linked_memory_ids 加权）；threshold 在融合**前**做语义门（低于阈值即使 BM25/实体可提升也排除）。
3. **"防幻觉前缀"工程防御**：LLM 输出被系统性地视为不可信——UUID 映射（`uuid_mapping` 防止 LLM 幻觉 id 引用）、hash 去重（md5）、阈值前置、JSON 解析 fallback 链（chatty LLM 测试）。
4. **OSS 漏斗产品化**：telemetry（默认 0.1 采样 + lifecycle 100% + `$identify` 永不丢）+ notices（GitHub raw 远程配置 + bundled fallback + TTL + A/B variant_split）+ MemoryClient 默认指向 `api.mem0.ai` 且 `user_id = md5(api_key)` + `_maybe_alias_anon_to_email`——**开源 SDK 是平台获客层，不是终点**。
5. **治理前置**：AGENTS.md 双门（PR 必须签 CLA + 链接 `accepted` issue，bot 一分钟关违反者）+ `.github/workflows/` 修改需维护者批准（publishing 凭据按文件名 pin）+ 包级工具链分治（polyglot monorepo 每包自定 lint 规则）。

## 与已有 Corpus 的连接
- **letta-code**（自编辑记忆 harness）：记忆来源对照——Letta 让 agent 主动编辑记忆 vs Mem0 自动抽取
- **hermes-agent**（MEMORY.md 冻结快照）：文件快照 vs 向量抽取
- **dynamo**（KV 缓存记忆）：推理侧原始 KV vs 语义侧压缩事实
- **dsh-memory-evolve**（本地记忆演进）：同类抽取式路线
- 对照结论：**Mem0 = "外部记忆即事实层"（LLM 压缩 + 向量召回）**，与 harness 内置记忆（letta/hermes）是互补路线，不是替代

## 质量指标
Facts 38 ｜ EK 38（≥80 边）｜ KO 9 ｜ Candidates 6 ｜ Flows 7 ｜ 反例 20+
