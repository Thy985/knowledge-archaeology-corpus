# Evidence Index — MemEvolve / EvolveLab

口径：S3 = 已实现（静态代码/配置可核）；S0-vendor = 仓库自述/论文声明，未复跑。
所有行号基于 commit `6035d5659d7a092dbfa6a87b1a32a3cee652ba54`。

## A. 接口与数据模型（E-001 ~ E-010）

| ID | 事实 | 来源 | 强度 |
|---|---|---|---|
| E-001 | 抽象基类只有 3 个抽象方法 `provide_memory` / `take_in_memory` / `initialize`，无 update/delete/merge/forget 一等操作 | `EvolveLab/base_memory.py:10-51` | S3 |
| E-002 | `MemoryStatus` 只有 `BEGIN` / `IN` 两个值，源码注释 “only these two status” | `EvolveLab/memory_types.py:11-15` | S3 |
| E-003 | `MemoryType` 共 13 个枚举值 | `EvolveLab/memory_types.py:18-39` | S3 |
| E-004 | `PROVIDER_MAPPING` 13 项，`MemoryType -> (ClassName, ModuleName)`，是唯一 dispatch 表 | `EvolveLab/memory_types.py:48-63` | S3 |
| E-005 | `MemoryItemType` 只有 `TEXT` / `API` | `EvolveLab/memory_types.py:66-69` | S3 |
| E-006 | `MemoryRequest` 字段：`query/context/status/additional_params` | `EvolveLab/memory_types.py:72-78` | S3 |
| E-007 | `MemoryItem` 字段：`id/content/metadata/score(optional)/type(默认 TEXT)` | `EvolveLab/memory_types.py:81-88` | S3 |
| E-008 | `MemoryResponse` 字段：`memories/memory_type/total_count/request_id` | `EvolveLab/memory_types.py:91-97` | S3 |
| E-009 | `TrajectoryData` 字段：`query/trajectory/result(optional)/metadata(optional)` | `EvolveLab/memory_types.py:100-105` | S3 |
| E-010 | `get_config()` 返回 `self.config.copy()`，外部拿到的是浅拷贝 | `EvolveLab/base_memory.py:53-59` | S3 |

## B. 检索与嵌入基础设施（E-011 ~ E-020）

| ID | 事实 | 来源 | 强度 |
|---|---|---|---|
| E-011 | provider 侧同时使用 `TfidfVectorizer` 与 `SentenceTransformer`（可选导入） | `EvolveLab/providers/agent_kb_provider.py:10-12`；`cerebra_fusion_memory_provider.py:24-30` | S3 |
| E-012 | 默认嵌入模型 `sentence-transformers/all-MiniLM-L6-v2`，缓存到 `./storage/models` | `agent_kb_provider.py:25-26`；`cerebra_fusion_memory_provider.py:79-80` | S3 |
| E-013 | 多个 provider 将 embedding 维度硬编码为 384 | `dilu_memory_provider.py:63`；`evolver_memory_provider.py:144-157`；`generative_memory_provider.py:56`；`voyager_memory_provider.py:65` | S3 |
| E-014 | Agent-KB 检索为 text+semantic 加权混合，默认权重 0.5/0.5 | `agent_kb_provider.py:302-305`；`197-268` | S3 |
| E-015 | Agent-KB 先用 LLM 把 query 改写为检索词，失败则回退原 query | `agent_kb_provider.py:333-360` | S3 |
| E-016 | Cerebra 检索为 text+semantic（权重 0.2/0.8），并有 `min_score` 阈值 | `cerebra_fusion_memory_provider.py:216-219`；`532-551` | S3 |
| E-017 | Cerebra 图扩展：语义边阈值 0.75、最多扩展 3 个邻居、可关闭 | `cerebra_fusion_memory_provider.py:221-224`；`435-483`；`493-531` | S3 |
| E-018 | Cerebra 在 IN 阶段把 `context` 拼进检索词并追加 “continuation resume partial progress” | `cerebra_fusion_memory_provider.py:907-908` | S3 |
| E-019 | Cerebra 用 `min_score_in_phase` 作为 IN 阶段阈值 | `cerebra_fusion_memory_provider.py:913` | S3 |
| E-020 | Expel / Dynamic-Cheatsheet / MemP 默认 `top_k=1` | `expel_provider.py:34`；`dynamic_cheatsheet_provider.py:87`；`memp_memory_provider.py:294` | S3 |

## C. Provider 行为矩阵（E-021 ~ E-040）

| ID | 事实 | 来源 | 强度 |
|---|---|---|---|
| E-021 | 仅 BEGIN 响应的 provider：agent_kb、agent_workflow、dilu、evolver、generative、memp、voyager（7 个） | 各文件 `MemoryStatus` 分支：`agent_kb_provider.py:371`；`agent_workflow_memory_provider.py:248`；`dilu_memory_provider.py:185`；`evolver_memory_provider.py:451`；`generative_memory_provider.py:176`；`memp_memory_provider.py:289`；`voyager_memory_provider.py:246` | S3 |
| E-022 | 仅 IN 响应的 provider：dynamic_cheatsheet | `dynamic_cheatsheet_provider.py:249` | S3 |
| E-023 | BEGIN+IN 均响应：cerebra、lightweight、expel、mobilee、skillweaver（5 个） | `cerebra_fusion_memory_provider.py:630-652`；`lightweight_memory_provider.py:214-242`；`expel_provider.py:198-216`；`mobilee_provider.py:173-189`；`skillweaver_provider.py:217-219` | S3 |
| E-024 | Agent-KB 在 BEGIN 合成 “Student Guidance + Teacher Guidance” 双角色提示 | `agent_kb_provider.py:428-448` | S3 |
| E-025 | Agent-KB 只摄入成功任务；成功判定接受 `is_correct/success/task_success`，缺失即 False | `agent_kb_provider.py:614-618`；`663-679` | S3 |
| E-026 | Agent-KB 每次摄入后重建 `AKB_Manager`（重新构建 TF-IDF 与 embedding 索引） | `agent_kb_provider.py:649-653`；`116-146` | S3 |
| E-027 | Agent-Workflow 只从成功轨迹归纳 workflow，且只在 BEGIN 提供 | `agent_workflow_memory_provider.py:203-282`；`286` | S3 |
| E-028 | Dilu 用 JSON 存储，LLM 摘要后入库，只在 BEGIN 提供 | `dilu_memory_provider.py:110-180`；`249-270` | S3 |
| E-029 | Generative 用 JSON 存储 + LLM 摘要，只在 BEGIN 提供 | `generative_memory_provider.py:99-171`；`275-300` | S3 |
| E-030 | Voyager 用 JSON 存储 + LLM 摘要，只在 BEGIN 提供 | `voyager_memory_provider.py:127-140`；`246`；`300-330` | S3 |
| E-031 | Evolver 是“原则库”：LLM 摘要成功/失败经验为 `[DESCRIPTION]` + `[STRUCTURE]` | `evolver_memory_provider.py:239-357` | S3 |
| E-032 | Evolver 打分公式 `(success_count+1)/(usage_count+2)`，默认剪枝阈值 0.2（config 覆盖 0.3） | `evolver_memory_provider.py:377-380`；`89-92`；`EvolveLab/config.py:102-109` | S3 |
| E-033 | Evolver 合并条件：cosine > 0.8 且 LLM 判定语义相同；否则新建 uuid4 记录 | `evolver_memory_provider.py:678-683`；`636-688` | S3 |
| E-034 | Evolver 写盘只保存 `memories`，不保存 embeddings；initialize 发现长度不符则全量重算 | `evolver_memory_provider.py:161-175`；`123-159` | S3 |
| E-035 | Evolver 仅 BEGIN 返回；IN 直接空响应 | `evolver_memory_provider.py:450-452` | S3 |
| E-036 | MemP 记忆 = 抽象 procedure script（<100 词）+ concrete summary（<150 词） | `memp_memory_provider.py:203-227`；`229-258` | S3 |
| E-037 | MemP 在任务失败时**改写**已检索到的记忆，并记录 `revision_count`，而非新增 | `memp_memory_provider.py:392-476` | S3 |
| E-038 | MemP 默认 `is_correct` 缺失时视为 `"true"`（默认成功） | `memp_memory_provider.py:480` | S3 |
| E-039 | Expel 同时维护 insights 与成功轨迹两个 JSON；TF-IDF 检索；成功/失败 insight 分型 | `expel_provider.py:32-34`；`85-110`；`220-247`；`348-371` | S3 |
| E-040 | Dynamic-Cheatsheet 为单份全局 cheatsheet（<200 词），按相似 query 由 LLM 重写 | `dynamic_cheatsheet_provider.py:80-87`；`293-321` | S3 |

## D. 双记忆与工具/技能记忆（E-041 ~ E-050）

| ID | 事实 | 来源 | 强度 |
|---|---|---|---|
| E-041 | Lightweight 分短期（任务内）与长期（跨任务）两层 | `lightweight_memory_provider.py:73-120`；`194-363` | S3 |
| E-042 | `enable_longterm_provision` 默认 False（类与 config 两处一致） | `lightweight_memory_provider.py:108-116`；`EvolveLab/config.py:44-53` | S3 |
| E-043 | 短期容量默认 15、注入间隔 5 步；长期 strategic/operational 各 30、buffer 20、top_k 3 | `lightweight_memory_provider.py:99-120`；`EvolveLab/config.py:44-53` | S3 |
| E-044 | 短期抽取用 context delta：current 以 last 为前缀则取增量，否则整段；delta<50 字符跳过 | `lightweight_memory_provider.py:558-582` | S3 |
| E-045 | 短期抽取由 LLM 产出 `{step_summary,key_extracts}`，最多 3 条，过滤系统/格式指令 | `lightweight_memory_provider.py:584-716` | S3 |
| E-046 | 短期超限时由 LLM 选保留项，失败回退 FIFO | `lightweight_memory_provider.py:799-920` | S3 |
| E-047 | 长期只从成功轨迹写入，每任务最多 2 strategic + 2 operational，长度<20 丢弃，SHA256 去重 | `lightweight_memory_provider.py:1057-1157`；`1466-1503` | S3 |
| E-048 | 长期剪枝仅在超过 max+buffer 时触发，优先 LLM 选删除项，失败回退 `success_count*2+usage_count` | `lightweight_memory_provider.py:1159-1341` | S3 |
| E-049 | `used_memory_ids` 形如 `strategic_3`，更新使用量时按下划线解析索引；剪枝后索引可能位移 | `lightweight_memory_provider.py:501-556` | S3（缺陷候选） |
| E-050 | Cerebra 工具体系：从轨迹抽取代码、安全检查、注册为 callable；工具候选走 embedding + LLM 路由 | `cerebra_fusion_memory_provider.py:955-981`；`1232-1355`；`1570-1585` | S3 |
| E-051 | SkillWeaver 把生成技能写成 `.py` 文件并加载 registry，返回可包装 tool；要求 `is_correct` 与 `task_success` 同时为真 | `skillweaver_provider.py:41-63`；`147-195`；`355-365` | S3 |
| E-052 | MobileE 维护 tips 与 shortcuts 两套 JSON，检索为词法 token 匹配 | `mobilee_provider.py:24-54`；`65-171` | S3 |
| E-053 | Cerebra 每 50 个任务做一次 consolidation（合并相似节点、剪边、优化权重） | `cerebra_fusion_memory_provider.py:233-234`；`1044-1048`；`1378-1503` | S3 |

## E. 元进化管线（E-054 ~ E-070）

| ID | 事实 | 来源 | 强度 |
|---|---|---|---|
| E-054 | 四阶段 Analyze→Generate→Create→Validate，由 `MemoryEvolver` 编排 | `MemEvolve/core/memory_evolver.py:152-473` | S3 |
| E-055 | 阶段完成状态持久化到 `state.json`，支持从中断处续跑 | `MemEvolve/core/memory_evolver.py:1-150` | S3 |
| E-056 | 分析阶段是 tool-using `AnalysisAgent`，工具为 TrajectoryViewer/StepViewer/MemoryDatabaseViewer，`max_steps=20` | `base_agent.py:134-159`；`MemEvolve/phases/phase_analyzer.py:15`；`51-91` | S3 |
| E-057 | 分析 Agent 自身 `memory_provider=None`（不被记忆增强） | `base_agent.py:152-159` | S3 |
| E-058 | 成功判定在 analyzer 中兼容 `judgement/score/is_correct/correct/success`，全缺失判 False | `MemEvolve/phases/phase_analyzer.py:138-199` | S3 |
| E-059 | 同一成功判定逻辑在 `trajectory_tools.py` 与多个 provider 中重复实现 | `MemEvolve/utils/trajectory_tools.py:55-84`；`388-430`；`agent_kb_provider.py:663-679` | S3 |
| E-060 | 生成 prompt 注入 `memory_types.py` 全文与目标 provider 源码全文 | `MemEvolve/phases/phase_generator.py:102-125`；`155-239` | S3 |
| E-061 | 生成温度 `0.3 + 0.9*creativity`；`max_tokens=60000`；原始输出落盘 | `MemEvolve/phases/phase_generator.py:241-303`；`EvolveLab/config.py` 对侧 `MemEvolve/config.py:44-56` | S3 |
| E-062 | 解析依赖正则：`**Class Name**` / `**Module Name**` / `**Enum Name**` / `**Enum Value**` / ```python 代码块 / `**key**: value` | `MemEvolve/phases/phase_generator.py:305-386` | S3 |
| E-063 | 创建顺序：写 provider 文件 → 改 `memory_types.py` enum/mapping → 校验 config_updates → 改 `config.py` | `MemEvolve/phases/memory_creator.py:86-284` | S3 |
| E-064 | 创建过程非原子：第 3 步失败时已写 provider/enum/mapping，无回滚 | `MemEvolve/phases/memory_creator.py:86-284` | S3 |
| E-065 | config_updates 仅做 `ast.literal_eval` + key 归一化；MemoryEvolver 对非法更新只打印警告并继续 | `MemEvolve/phases/memory_creator.py`；`MemEvolve/core/memory_evolver.py:266-271` | S3 |
| E-066 | 静态检查是字符串包含检查（`BaseMemoryProvider`、三个方法名、import 文本），非语义 | `MemEvolve/phases/phase_validator.py:224-293` | S3 |
| E-067 | 验证器 docstring 称 “without automatic fixes”，但构造默认 `enable_auto_fix=True` | `MemEvolve/phases/phase_validator.py:29-43` | S3（文档/实现矛盾） |
| E-068 | 失败时调用 `SWEAgentValidator.fix_memory_system`，修复后 `_sync_to_real_environment` 用 copy2 覆盖真实 provider 文件 | `MemEvolve/phases/phase_validator.py:197-214`；`814-835` | S3 |
| E-069 | 快检在 `EvolveLab` + `storage` 的隔离副本中执行：清 sys.modules、按 mapping 导入、注入 model、`initialize()==True`、BEGIN 类型校验、`take_in_memory` 成功 | `MemEvolve/phases/phase_validator.py:332-371`；`433-790` | S3 |
| E-070 | 隔离通过 sys.path 修改与模块缓存清理实现；仅当 verdict=passed 才清理隔离目录 | `MemEvolve/phases/phase_validator.py:438-443`；`473-482`；`804-810` | S3 |

## F. 多轮搜索、评测与运行（E-071 ~ E-090）

| ID | 事实 | 来源 | 强度 |
|---|---|---|---|
| E-071 | AutoEvolver 每轮 Step0 把 `./storage/{provider}` 重命名为时间戳备份，令 provider 从空存储开始（Fairness） | `MemEvolve/core/auto_evolver.py:812-840` | S3 |
| E-072 | 每轮先在 x 个任务上跑基座 provider | `MemEvolve/core/auto_evolver.py:788-1008`；`MemEvolve/config.py:52` | S3 |
| E-073 | 生成 m 个候选系统（默认 3），每个系统独立重跑 analysis | `MemEvolve/core/auto_evolver.py:861-892` | S3 |
| E-074 | 基座 + validated 系统在同一 batch 上并行评估 | `MemEvolve/core/auto_evolver.py:894-918` | S3 |
| E-075 | 默认选择：accuracy 降序、tokens 升序；Pareto 可选（0.6 accuracy / 0.25 token / 0.15 time） | `MemEvolve/core/auto_evolver.py:623-782` | S3 |
| E-076 | Finalists 在 `random.sample(batch, y)` + 新 x 任务上复评，winner 写 state；`dataset_cursor += 2x` | `MemEvolve/core/auto_evolver.py:920-945` | S3 |
| E-077 | checkpoint 每步落盘，支持中断续跑；每轮生成数按已 validated 数量扣减 | `MemEvolve/core/auto_evolver.py:788-1008` | S3 |
| E-078 | Runner 以子进程调用，`concurrency=1`，无 timeout；进程非 0 退出即抛错 | `MemEvolve/utils/run_provider.py:23-86` | S3 |
| E-079 | 任务索引 0-based 传入后转为 1-based CLI 参数 | `MemEvolve/utils/run_provider.py:38` | S3 |
| E-080 | 每个线程创建独立 provider 实例（避免共享状态） | `run_flash_searcher_mm_gaia.py:594-607` | S3 |
| E-081 | 任务结束后以 `TrajectoryData` 摄取；成功任务与异常任务都调用 `take_in_memory`（异常时 `is_correct=False`） | `run_flash_searcher_mm_gaia.py:319-338`；`369-388` | S3 |
| E-082 | 正确性由 LLM judge 产生；judge 异常时 `is_correct` 保持 False，无 unknown 状态 | `run_flash_searcher_mm_gaia.py:303-317` | S3 |
| E-083 | 评测指标聚合：accuracy 为 1/0 均值，tokens 取 total/prompt/completion 均值 | `MemEvolve/utils/trajectory_tools.py:116-230` | S3 |
| E-084 | Agent 运行时在 planning 与每个 step 注入 memory guidance（user role，包裹于 `————Memory System Guidance————`） | `FlashOAgents/agents.py:413-425`；`782-795` | S3 |
| E-085 | 记忆注入发生在 planning 前与每步一次，而非工具调用 | `FlashOAgents/agents.py:414`；`783` | S3 |
| E-086 | `MemoryStatus.IN` 请求携带 `context` 与 `step_number` | `FlashOAgents/agents.py:783`；`EvolveLab/memory_types.py:72-78` | S3 |
| E-087 | CLI `evolve_cli.py` 暴露 analyze/generate/create/validate/auto 子命令 | `evolve_cli.py:1-120` | S3 |
| E-088 | 数据集映射：gaia / webwalkerqa / xbench / taskcraft / coin_flip 与各自 runner | `MemEvolve/config.py:16-31` | S3 |
| E-089 | 进化默认参数：x=20, t=2, y=5, m=3, seed=42, analysis_max_steps=20, creativity=0.5 | `MemEvolve/config.py:44-56` | S3 |
| E-090 | 13 个 provider 中 **11 个**以本地 JSON 文件持久化；SkillWeaver 写 `.py` 技能文件，Dynamic-Cheatsheet 写 `.txt` 全局 cheatsheet | 各 provider `_save_*`：`agent_kb:649`；`agent_workflow:154`；`cerebra:409`；`dilu:125`；`dynamic:80-87`（`.txt`）；`evolver:171`；`expel:370/447`；`generative:114`；`lightweight:1462`；`memp:128`；`mobilee:386/404`；`skillweaver:41`（`.py`）；`voyager:127` | S3 |

## G. 仓库治理、测试与文档（E-091 ~ E-100）

| ID | 事实 | 来源 | 强度 |
|---|---|---|---|
| E-091 | `EvolveLab/` 与 `MemEvolve/` 无任何测试文件；仅 `mini-swe-agent/tests` 存在 30 个测试文件 | `find` 全仓结果 | S3 |
| E-092 | README 声称 ICML'26 接收、arXiv 2512.18746 | `README.md` | S0-vendor |
| E-093 | README 声称 11 基线 + 2 evolved = 13；`MemoryType` 实际 13 项 | `README.md`；`memory_types.py:18-39` | S0-vendor / 代码侧 S3 |
| E-094 | README 明确 “not the final release” | `README.md` | S0-vendor |
| E-095 | README 数据集基准日期：GAIA Feb 2025、WebWalkerQA 170 子集、xBench DeepSearch | `README.md` | S0-vendor |
| E-096 | 论文自报成绩（SmolAgent / Flash-Searcher 最高 +17.06%，跨基准 +2.0~9.09%） | `README.md` | S0-vendor（未复跑） |
| E-097 | `base_provider_template.py` 是明确占位骨架：默认 `MemoryType.AGENT_KB`、provide 返回空、take_in 直接成功、多处 TODO | `EvolveLab/providers/base_provider_template.py:27-172` | S3 |
| E-098 | 13 个 provider 代码规模从 296 行（Dilu）到 1,601 行（Cerebra），中位数约 447 行 | `wc -l` 输出 | S3 |
| E-099 | 仓库自带数据目录 `data/gaia`、`data/webwalkerqa`，但未验证数据文件完整性/可运行性 | 目录结构 | S3（存在性） |
| E-100 | 仓库为 vendored 多子项目聚合体：`FlashOAgents`、`mini-swe-agent`、`model_eval` 与 MemEvolve 并存 | 顶层结构 | S3 |

**计数**：100 条。S3 = 96 条；S0-vendor = 4 条（E-092~E-096 中 4 项论文自述）。无 S4/S5。
