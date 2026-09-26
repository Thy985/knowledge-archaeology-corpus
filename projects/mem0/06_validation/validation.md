# 06 Validation & Evidence — Mem0（ARCH-2026-09-21-001）

## 6.1 验证矩阵（先盲重建后对比——本 run 考古为单 Agent 顺序采集 + 审计交叉复核）

### Truth（Source Fidelity）
| 检查项 | 结果 | 说明 |
|---|---|---|
| 版本 2.1.0 | PASS | pyproject.toml:7 直接读取 |
| commit a39a802 | PASS | git log 实测 |
| ADD 管道六阶段 | PASS | main.py:879-976 逐行 |
| scoring 加性融合 | PASS | scoring.py:60-130 全文 |
| telemetry 0.1 采样 | PASS | telemetry.py:30-48 + test_telemetry_sampling |
| notices 全 disabled | PASS | oss_notices_config.json 实测（enabled:false ×7） |
| AGENTS.md 双门 | PASS | AGENTS.md 原文（CLA + accepted label + bot 1 分钟） |
| server JWT/bcrypt | PASS | server/auth.py 原文 |
| procedural 短路 | PASS | main.py:815-823 |
| 身份键不可变 | PASS | main.py:2050 `_strip_identity_keys` |

### Coverage
| 面 | 覆盖 | 未覆盖（Candidates） |
|---|---|---|
| 写入路径（add/procedural/raw） | 完整 | — |
| 检索路径（search/hybrid/entity） | 完整 | — |
| 更新/删除（update/delete/delete_all） | 完整 | — |
| 遥测/通知 | 机制级 | 迁移信号用途（C-04） |
| server | 认证/限流/治理 | Memory 类集成方式（C-07） |
| 版本口径 | 表面 | v2→v3 迁移细节（C-01） |
| 集成插件 | 目录级 | 插件内部实现（16 个未逐一读） |
| 测试体系 | 命名/重点 | 129 文件未全读（按风险抽样） |

### Causality
- EK-01（单遍抽取）是 EK-02（hash 去重）的触发者：Phase 2 产出 → Phase 4 消费 ✅
- EK-12（threshold 门）先于 EK-09（融合）：scoring.py:96-99 在 score_and_rank 内部 ✅
- EK-17（身份不可变）约束 EK-18（实体清理）：update 路径内 `_strip_identity_keys` 先于实体重链接 ✅
- KO-04（漏斗）的因果链是推断（telemetry→notices→client 指向平台），标 Hypothesis（C-02）✅

### Flow 验证
- F-1 add 全链每个 symbol 可回溯（main.py 行号）✅
- F-5 server 认证链每个函数可回溯（auth.py）✅
- F-2 状态机：delete 后 is_deleted=1 且向量删除+实体清理（main.py:2116-2126）✅
- F-7 Policy：notices 拉取失败→bundled fallback（notices.py:30-40）✅

### Abstraction
| KO | 判定 | 依据 |
|---|---|---|
| KO-01 三段式 | L3 合理 | 单项目强证据，跨项目对照已在写（letta/hermes 均可映射） |
| KO-02 三信号互补 | L4 合理 | scoring 机制跨 RAG 领域通用；dynamo 双检索对照 |
| KO-03 LLM 防御链 | L3 合理 | 单项目机制簇，未跨项目验证 → 不升 L4 |
| KO-04 漏斗产品化 | L4 有条件 | Mem0 内证据链完整；跨项目验证 pending（C-06b） |
| KO-05~09 | L3 合理 | 各为机制簇/主题簇，未达跨项目 Principle |

### Counterexample（反例预算：每个 L3+ KO ≥3 个定向攻击）
| KO | 反例 | 结果 |
|---|---|---|
| KO-01 三段式 | infer=False 路径无 LLM 压缩（main.py:881）——写入也可不走压缩 | 收紧：KO-01 限定"infer=True 默认路径" |
| KO-01 | 无事实时 save_messages 但向量库无写入（main.py:909-912） | 确认：兜底不等于落库 |
| KO-02 三信号 | keyword_search 不支持的 store（main.py:548-556）混合禁用 | 收紧：三信号是"能力允许时"的融合 |
| KO-02 | rerank 开启时 reranker.rerank 覆盖原始结果（main.py:1468-1473） | 补充：第四信号（rerank）存在但默认关 |
| KO-03 防御链 | extract_json 失败时静默 []（main.py:905-916）——防御链有终点 | 确认：防御链末端是丢数据（诚实但损失） |
| KO-03 | md5 冲突理论可能静默漏记忆（无冲突处理） | Candidate C-05 |
| KO-04 漏斗 | notices 全 disabled——机制在但未激活 | 收紧：漏斗机制是"就绪态"非"激活态" |
| KO-05 工厂 | LlmFactory 用 provider_to_class 显式映射，非动态 import 全发现 | 确认：扩展需改表（显式注册） |
| KO-06 诚实失败 | `_create_procedural_memory` 空响应 raise ValueError（main.py:2019-2023） | 确认：同类诚实失败 |
| KO-07 事件溯源 | GET 无事件（仅 telemetry capture）——读取不溯源 | 收紧：仅变更事件 |
| KO-08 治理 | AUTH_DISABLED 逃生阀存在——运行时治理可被 env 绕过 | 收紧：治理含显式降级路径 |
| KO-09 scope | add 接受顶层 user_id/agent_id/run_id，search 拒绝——API 分裂 | 收紧：契约不一致是 v2 历史遗留（代码注释明示） |

### Epistemic 检查
| 对象 | 标注 | 诚实性 |
|---|---|---|
| 全部 EK | L1/L2 | 每条带证据行号 ✅ |
| KO-01~09 | L3/L4 | KO-04 标"跨项目验证 pending" ✅ |
| C-01~07 | Candidate | 显式区分 Observation/Hypothesis ✅ |
| 无"Principle/Law"级声明 | — | 单项目证据不足不升 ✅ |

## 6.2 反例清单（Counterexamples，保留）
1. `infer=False`：raw 写入绕过 LLM 压缩（main.py:881-907）
2. keyword_search 不支持 → 混合降级语义-only（main.py:548-556）
3. rerank=True 且 reranker 可用 → 覆盖融合结果（main.py:1468-1473）
4. extract_json 全失败 → 静默空抽取（main.py:905-916）
5. procedural memory 走摘要式而非 ADD-only（main.py:1993-2037）
6. 无事实抽取 → save_messages 但零向量写入（main.py:909-912）
7. batch 接口失败 → 逐条 fallback（embed/insert/history 三处）
8. delete_all 重复批次 → seen_batches 防死循环（main.py:1890-1944）
9. AUTH_DISABLED → 认证全绕过（server/auth.py）
10. timestamp/reference_date → ValueError（OSS 桩，main.py:775/1411）
11. A/B variant_split=0.5 → notices 变体分流（notices.py）
12. 高级过滤运算符处理 → 逻辑键从 effective_filters 移除（main.py:1524-1627）
13. MemoryClient 自定义 httpx.Client → base_url/headers 被强制覆盖（client/main.py:224-232）
14. memory 无 data → 检索结果跳过（main.py:1713-1715）
15. update data 参数 deprecated → 警告 + 兼容（main.py:1822-1828）
16. _safe_deepcopy_config → deepcopy 失败手工拷贝（main.py:270-300）
17. entity boost embed_batch 长度不符 → 跳过 boost（main.py:1750-1755）
18. rerank 异常 → 保留原始结果（main.py:1468-1473）
19. storage batch_add_history 失败 → 逐条 add（main.py:970-975）
20. 重复批次日志警告后停止 delete_all（main.py:1907-1910）

## 6.3 Contradictions（记录，无隐藏）
| # | 矛盾 | 裁决 |
|---|---|---|
| CT-1 | 版本号 2.1.0 vs 代码 "V3 PIPELINE" 注释 vs 候选卡 "v3.0 算法" | 未闭合——OSS SDK 版本与平台 v3 命名的关系留 Candidate C-01 |
| CT-2 | add 接受顶层 scope 参数 vs search 拒绝（docstring 明示此分裂） | 代码注释确认是有意 API 分裂（v2 遗留） |
| CT-3 | MemoryClient 默认 api.mem0.ai 但 OSS Memory 完全本地 | 双形态设计，非矛盾 |

## 6.4 Reconciliation（本 run 无需跨 Agent 冲突——单执行者；按历史流程保留审查建议）
- R-1: KO-01/KO-02/KO-04 已按反例收紧（见上表"收紧"列）
- R-2: C-01/C-04/C-07 明确标注"未闭合"，不冒充事实
- R-3: 无"同子系统=聚合理由"假聚合（aggregation_rule 全部声明）

## 6.5 质量指标
| 指标 | 值 |
|---|---|
| Facts（Project Layer） | 41 |
| EK | 38（34 活跃 + 4 D 级；≥80 边；0 游离） |
| KO | 9（L3×7 / L4×2） |
| Candidates | 7 |
| Flows | 7（全 symbol 可回溯） |
| 反例 | 20 |
| Contradictions | 3（含 1 未闭合） |
| Epistemic 混淆 | 0 |
