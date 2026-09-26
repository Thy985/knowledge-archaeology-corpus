# Independent Validation Report — ARCH-2026-09-21-001 (mem0)

> Auditor 独立盲重建：不把考古结果当事实来源，独立重读仓库建立 Independent Findings 后对比。
> 抽查清单：uuid_mapping / DELETE_ALL_BATCH_SIZE / _strip_identity_keys / AGENTS.md bot / scoring threshold 前置 / internal_limit / entity boost 0.5 / AsyncMemory V3 管线 / notices variant_split / _payload_is_expired / enable_vision / MemoryBase / test_session_scope / 适配器数量。

## 判定统计
| 判定 | 数量 | 明细 |
|---|---|---|
| CONFIRMED | 14 | 见 6.1 抽查矩阵 |
| PARTIALLY_CONFIRMED | 1 | ID-1（async 同构未显式标注） |
| DOWNGRADED | 0 | — |
| OVER_GENERALIZED | 2 | ID-3（KO-02 跨项目对照）、ID-4（KO-04 L4 条件） |
| MISSING | 1 | ID-2（enable_vision 视觉消息） |
| CONTRADICTED | 0 | — |
| NEEDS_HUMAN_REVIEW | 1 | ID-5（C-01 版本口径） |

## Independent Findings（Auditor 独立观察，先于对比）

### ID-1（PARTIALLY_CONFIRMED）AsyncMemory 同构 V3 管线
- **观察**: AsyncMemory.add（main.py:2172+）同样走 V3 PHASED BATCH PIPELINE：`asyncio.to_thread` 包装 db/embedding/vector_store 阻塞调用（main.py:2580-2620），uuid_mapping 同构（2597-2599），ADDITIVE_EXTRACTION_PROMPT + AGENT_CONTEXT_SUFFIX 同构（2610-2620）。
- **影响**: 考古 EK-01 只写 sync 路径——事实正确但覆盖可加宽。**修正**: EK-01 links 或 EK 文本补一句 async 同构。

### ID-2（MISSING）enable_vision 视觉消息分支
- **观察**: `add()` 中 `if self.config.llm.config.get("enable_vision"): messages = parse_vision_messages(messages, self.llm, self.config.llm.config.get("vision_details"))`（main.py:864-867；async 2518-2519）。视觉输入（图片）→ LLM 描述 → 进抽取管道。
- **影响**: EK 层未收录"视觉消息解析"机制——这是记忆抽取的独立输入面（多模态记忆）。**修正**: 补 EK-36（视觉消息解析）。

### ID-3（OVER_GENERALIZED）KO-02 跨项目对照误引 dynamo
- **观察**: KO-02 写"dynamo 的 KV+语义双检索同理"——dynamo（ai-dynamo/dynamo）是推理 KV 缓存（原始激活存取），**不是**语义混合检索。
- **影响**: 对照引用错误会误导跨项目认知。**修正**: 删除 dynamo 对照，改为"跨项目验证 pending（需 RAG/记忆类项目）"。

### ID-4（OVER_GENERALIZED，轻微）KO-04 L4 有条件
- **观察**: KO-04（OSS 漏斗产品化）升 L4，证据链在 Mem0 内完整（telemetry→notices→client→迁移信号），但"商业化开源通用架构模式"解释范围扩大依赖单项目。
- **影响**: 单项目 L4 需显式 pending 标注。**修正**: 已在 KO-04 标"跨项目验证 pending"，Validation 中重申。

### ID-5（NEEDS_HUMAN_REVIEW）版本口径
- **观察**: pyproject 2.1.0 vs 代码 "V3 PIPELINE" 注释 vs 候选卡 "v3.0.0 新算法" vs `generate_additive_extraction_prompt` 注释 "Ported from platform/backend/shared/core/utils/prompt_builder.py"。
- **影响**: OSS SDK 版本号与平台 v3 命名的关系无法从仓库内完全闭合。**修正**: 保留 Candidate C-01；建议 Owner 人工确认或读 docs/migration/oss-v2-to-v3。

## 6.1 抽查矩阵（CONFIRMED）
| # | Claim | 独立验证 |
|---|---|---|
| C-01 | uuid_mapping 防幻觉映射 | main.py:935-937（sync）+ 2597-2599（async）✅ |
| C-02 | DELETE_ALL_BATCH_SIZE 循环删除 | main.py:136（=1000）+ 1924 ✅ |
| C-03 | _strip_identity_keys 身份不可变 | main.py:143-164（同值静默/改值警告）✅ |
| C-04 | AGENTS.md bot 一分钟关 PR | AGENTS.md:13（"A bot closes it within a minute"）✅ |
| C-05 | threshold 前置门控 | scoring.py:96-99（语义分先过滤再融合）✅ |
| C-06 | internal_limit = max(limit*4, 60) | main.py:1641 ✅ |
| C-07 | entity boost 0.5 阈值 | main.py:1738/1793 ✅ |
| C-08 | telemetry 0.1 采样 + lifecycle 100% | telemetry.py:30-48 + test_telemetry_sampling ✅ |
| C-09 | notices variant_split 0.5 | notices.py:59/92 ✅ |
| C-10 | 适配器 21/15/28/7 | llms 21 / embeddings 15 / vector_stores 28 / reranker 7 实测 ✅ |
| C-11 | _payload_is_expired 异常安全 | main.py:442-456（ValueError→False）✅ |
| C-12 | MemoryClient md5(api_key) | client/main.py:219-220 ✅ |
| C-13 | server dummy_verify 防时序 | server/auth.py:28-31 ✅ |
| C-14 | 混合检索三信号 + max_possible 分档 | scoring.py:60-130 全文 ✅ |

## 6.2 专项攻击（SOP 指定重点）
| 攻击面 | 结果 |
|---|---|
| 单案例→Pattern | KO-01/03/05-09 均单项目强证据标 L3；未冒充跨项目原则 ✅ |
| Pattern→L4 | KO-02（三信号）L4 有误对照（ID-3 修正后仍 L4，跨项目 pending）；KO-04 L4 有条件 ✅ |
| 项目经验→通用 Principle | 无 Principle/Law 级声明 ✅ |
| ADR→实现事实 | 无 ADR 文档；实现事实全部来自代码（无文档冒充）✅ |
| Flow Edge 真实性 | F-1~F-7 逐 symbol 抽查可回溯（抽查 C-01~C-14 覆盖 F-1/2/3/5/6/7 关键 Edge）✅ |
| bypass 路径 | AUTH_DISABLED（F-5 已列）；infer=False 绕过 LLM（F-1 已列）；A/B variant_split（F-7 已列）✅ |
| override 路径 | MemoryClient 自定义 client 的 base_url/headers 被强制覆盖（client/main.py:224-232，考古反例 13）✅ |
| exception 路径 | _payload_is_expired ValueError→False；embed/insert/history 三处 batch→单条 fallback ✅ |
| alternate path | procedural memory 短路（main.py:815-823）；_create_procedural_memory 摘要式 vs ADD-only ✅ |
| direct call | Memory.add→_add_to_vector_store 直接调用（无间接层）✅ |
| admin path | server /configure POST require_admin ✅ |
| fallback | notices 远程失败→bundled；StaticFlagResult 无 PostHog 降级 ✅ |
| legacy path | update 的 data 参数 deprecated 兼容（main.py:1822-1828）；version=v1.1 配置默认值 ✅ |
| Epistemic 混淆 | Fact/Observation/Hypothesis 分层清楚；C-02/C-04/C-05/C-06 明确标注 ✅ |

## 6.3 必须回答的问题

### 原 Archaeology 最重要的 3 个成功
1. **V3 单遍 ADD-only 抽取管道的精确捕捉**（含 UUID 映射防幻觉、hash 去重、批量 fallback 三件套）——深度阅读才能发现的机制，且 sync/async 双确认。
2. **混合检索评分语义的准确还原**（threshold 前置 + max_possible 分档 + BM25 长度自适应）——scoring.py 全文无错读。
3. **OSS 漏斗产品化链路的完整识别**（telemetry 采样 → notices 远程配置 → MemoryClient 匿名身份别名 → 迁移信号）——这是 Mem0 最独特的认知增量。

### 最重要的 3 个错误
1. **KO-02 跨项目对照误引 dynamo**（KV 缓存非混合检索）——对照引用未验证，已修正。
2. **enable_vision 视觉消息分支遗漏**（main.py:864-867）——输入面覆盖缺口，补 EK-36。
3. **AsyncMemory 同构管线未显式标注**——EK-01 覆盖加宽，非事实错误。

### 是否存在关键遗漏
- 有 1 个机制级遗漏（enable_vision），已补 EK-36。
- 覆盖级未闭合：16 集成插件内部实现未逐一读（Candidate C-07）；server 与 OSS Memory 的集成方式未追踪（Candidate C-07）。

### 是否存在错误升维
- KO-02 L4 跨项目对照有误（ID-3）——升维本身保留但对照修正 + 明确 pending。
- KO-04 L4 有条件（ID-4）——pending 标注重申。
- 其余 KO 均 L3，无过度升维。

### 是否存在事实错误
- 无。14/14 抽查 CONFIRMED。

### 是否存在 Flow 错误
- 无。F-1~F-7 关键 Edge 全部可回溯实际代码。

### 是否发现新的 Benchmark / Regression Case
| # | 案例 | 来源 | 建议用途 |
|---|---|---|---|
| B-1 | LLM 输出 JSON 解析容错（纯 JSON/代码块/对话包装） | test_chatty_llm_parsing.py | skill 的"LLM 输出净化"基准 |
| B-2 | 混合检索 threshold 前置门控 vs 融合后过滤 | scoring.py:96-99 | 评分语义回归测试 |
| B-3 | delete_all 重复批次防死循环 | main.py:1890-1944 | 批量删除健壮性基准 |
| B-4 | 认证时序防枚举（dummy_verify） | server/auth.py:28-31 | 认证安全基准 |
| B-5 | 遥测采样分层（0.1 vs lifecycle 100%） | test_telemetry_sampling.py | 遥测策略基准 |

## 6.4 结论
**判定**: 考古结果总体可信（14 CONFIRMED / 1 PARTIAL / 0 CONTRADICTED / 0 事实错误）。需 2 处产物修正（ID-2 补 EK-36、ID-3 修正 KO-02 对照）+ 1 处加宽（ID-1 async 注记）。无需重做考古。
