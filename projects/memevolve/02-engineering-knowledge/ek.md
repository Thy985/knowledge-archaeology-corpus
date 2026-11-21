# Engineering Knowledge（EK Graph 节点）— 50 条

> 口径：每条 EK 都是**项目级**工程知识（L1/L2），不是跨项目原则。`links` 使用六类边：mechanism / subsystem / causal / dependency / constraint / contrast。
> `Strength` 指证据强度（S3 = 已实现静态可核）；`Epistemic` 区分 Fact / Observation。

---

## A. 接口与契约（EK-01 ~ EK-06）

### EK-01 记忆抽象被压缩为“取 / 喂 / 初始化”三段式
- **Layer**: L1 · **Type**: ARCHITECTURE · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: `BaseMemoryProvider` 只定义 `provide_memory`、`take_in_memory`、`initialize` 三个抽象方法，没有 update/delete/merge/forget 的接口级能力；所有“修改型”行为必须塞进 `take_in_memory` 内部。
- **Evidence**: E-001, E-010
- **Links**: `mechanism:EK-33`（写入形态被迫内聚）；`constraint:EK-02`（阶段契约同样被压缩）；`contrast:EK-25`（SkillWeaver 把“技能”也塞进同一接口）
- **Note**: 这是全项目最强的结构约束：13 个实现都在同一个窄接口下做不同的事。

### EK-02 记忆生命周期只有 BEGIN / IN 两个阶段状态
- **Layer**: L1 · **Type**: MEMORY · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: `MemoryStatus` 仅有 `BEGIN="begin"`、`IN="in"`，源码注释明确 “only these two status”；不存在 END/FAILED/DONE 阶段。任务结束的写入不是通过阶段枚举，而是 runner 直接调用 `take_in_memory`。
- **Evidence**: E-002, E-081
- **Links**: `causal:EK-08`（两阶段决定了注入点只有 planning 与每步）；`constraint:EK-07`（阶段支持矩阵由此产生）；`contrast:EK-05`（TrajectoryData 才是结束态的载体）
- **Note**: 因此“任务结束”在协议层没有显式信号，只能靠调用方约定。

### EK-03 13 套记忆系统由一张 enum→(class, module) 映射表统一调度
- **Layer**: L1 · **Type**: ARCHITECTURE · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: `MemoryType` 13 个枚举值与 `PROVIDER_MAPPING` 13 项一一对应；runner 通过 `MemoryType(value)` 查表并动态 import；新增系统必须同时改 enum、mapping、config 与 providers 目录。
- **Evidence**: E-003, E-004, E-063
- **Links**: `dependency:EK-39`（Creator 依赖锚点改这张表）；`constraint:EK-42`（验证器静态检查要求类/方法名出现）；`subsystem:EK-07`
- **Note**: 注册表是“权威流”的唯一入口（见 Flow Atlas Authority Flow）。

### EK-04 记忆载荷只有 TEXT 与 API 两种类型
- **Layer**: L1 · **Type**: MEMORY · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: `MemoryItemType = {TEXT, API}`。TEXT 被拼进提示词，API 携带 `wrapped_tool`/`callable` 供 Agent 直接调用（Cerebra、SkillWeaver 实际使用）。
- **Evidence**: E-005, E-050, E-051
- **Links**: `mechanism:EK-26`（Cerebra 同时返回两类）；`mechanism:EK-25`（SkillWeaver 只返回 API）；`contrast:EK-23`（Dynamic-Cheatsheet 只返回 TEXT）
- **Note**: API 型记忆无法被 JSON 序列化审计，导致“记忆内容”与“可执行代码”混在同一抽象里。

### EK-05 TrajectoryData 是唯一的摄取信封，正确性靠 metadata 约定
- **Layer**: L1 · **Type**: DATA · **Strength**: S3 · **Epistemic**: Fact · **Value**: B
- **Statement**: 所有 provider 的 `take_in_memory` 都接收 `TrajectoryData{query, trajectory, result, metadata}`；runner 写入 `{task_id,status,is_correct,full_query}`；`metadata` 无 schema 校验。
- **Evidence**: E-009, E-081, E-082
- **Links**: `causal:EK-20`（MemP 依赖 is_correct 决定 revise）；`causal:EK-31`（Lightweight 依赖它决定是否写长期）；`dependency:EK-34`
- **Note**: 正确性数据来自 LLM judge，judge 异常时是 False 而非 unknown。

### EK-06 配置只提供浅拷贝，嵌套结构仍可被外部改写
- **Layer**: L1 · **Type**: CONFIG · **Strength**: S3 · **Epistemic**: Fact · **Value**: C
- **Statement**: `get_config()` 返回 `self.config.copy()`；顶层键隔离，嵌套 dict/list 仍共享引用。
- **Evidence**: E-010
- **Links**: `constraint:EK-39`（Creator 改 config 的另一种路径）；`subsystem:EK-50`
- **Note**: 对 provider 内部状态的保护是不完整的。

---

## B. 阶段策略与检索（EK-07 ~ EK-13）

### EK-07 13 个 provider 的阶段支持呈 7 / 1 / 5 分布
- **Layer**: L2 · **Type**: MEMORY · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: 7 个 provider 仅在 BEGIN 返回（agent_kb、agent_workflow、dilu、evolver、generative、memp、voyager）；1 个仅 IN（dynamic_cheatsheet）；5 个双阶段（cerebra、lightweight、expel、mobilee、skillweaver）。IN 阶段是 Agent 每步执行时的唯一记忆入口，因此 7/13 的系统在任务中途“失忆”。
- **Evidence**: E-021, E-022, E-023
- **Links**: `causal:EK-08`；`contrast:EK-27`（Cerebra 双阶段且 IN 有独立阈值）；`mechanism:EK-14`
- **Note**: 这不是实现 bug，而是设计选择；但 README/论文把它们并列为“13 个记忆系统”，会掩盖阶段能力差异。

### EK-08 记忆以 user 角色消息注入，不是工具调用也不是系统提示
- **Layer**: L1 · **Type**: AGENT · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: `FlashOAgents/agents.py` 在 planning 前把 BEGIN 返回内容包成 `————Memory System Guidance————...————End Memory————` 的 **USER** 消息插入 `input_messages`；每个 step 把 IN 返回内容以同样格式追加到 `memory_messages`。
- **Evidence**: E-084, E-085, E-086
- **Links**: `causal:EK-02`；`dependency:EK-09`；`contrast:EK-04`（API 型记忆是唯一绕过提示词注入的路径）
- **Note**: 记忆被当作“用户说的话”，与真实用户消息同权；模型无法区分来源。

### EK-09 检索底座统一为 all-MiniLM-L6-v2 的 384 维嵌入，部分混合 TF-IDF
- **Layer**: L1 · **Type**: ENGINEERING · **Strength**: S3 · **Epistemic**: Fact · **Value**: B
- **Statement**: Agent-KB/Cerebra 等默认加载 `sentence-transformers/all-MiniLM-L6-v2` 缓存到 `./storage/models`；Dilu/Evolver/Generative/Voyager 硬编码 384 维；Agent-KB/Cerebra/Expel 叠加 TF-IDF。
- **Evidence**: E-011, E-012, E-013, E-014, E-016
- **Links**: `mechanism:EK-10`；`subsystem:EK-12`；`dependency:EK-18`（Evolver 的 384 维数组）
- **Note**: 13 个系统共享同一嵌入模型，降低了“实现差异”的解释力：差异主要来自写入与排序策略，而非表示能力。

### EK-10 混合检索的权重与阈值是 provider 级硬编码策略
- **Layer**: L2 · **Type**: ENGINEERING · **Strength**: S3 · **Epistemic**: Fact · **Value**: B
- **Statement**: Agent-KB 用 text/semantic 0.5/0.5；Cerebra 用 0.2/0.8，`min_score=0.22`，IN 阶段另用 `min_score_in_phase`；这些值写在构造函数/配置默认里，无自适应。
- **Evidence**: E-014, E-016, E-019
- **Links**: `contrast:EK-11`（Agent-KB 先改写 query，Cerebra 先扩图）；`causal:EK-27`
- **Note**: 阈值策略直接决定“召回多少条”，但没有任何评测驱动其调参。

### EK-11 Agent-KB 在检索前用 LLM 重写 query，形成“检索词生成→混合检索”两段式
- **Layer**: L2 · **Type**: ENGINEERING · **Strength**: S3 · **Epistemic**: Fact · **Value**: B
- **Statement**: `_reason_for_retrieval` 先让 LLM 产出检索意图/关键词，失败回退原 query；再用 TF-IDF + embedding 混合检索。
- **Evidence**: E-015
- **Links**: `mechanism:EK-13`；`contrast:EK-10`；`dependency:EK-09`
- **Note**: 这是 13 个系统中唯一显式的 query rewriting 步骤。

### EK-12 top_k 在 1~5 之间分散，且 MemP 硬编码为 1
- **Layer**: L1 · **Type**: CONFIG · **Strength**: S3 · **Epistemic**: Fact · **Value**: B
- **Statement**: Agent-KB 3、Cerebra 5、Dynamic 1、Evolver 1、Expel 1、Lightweight 长期 3；MemP 在 `provide_memory` 内部直接固定 top_k=1，不读配置。
- **Evidence**: E-012, E-020, `memp_memory_provider.py:294`
- **Links**: `contrast:EK-10`；`constraint:EK-07`（阶段能力决定 top_k 是否有意义）
- **Note**: “记忆条数”不是统一预算，无法做跨系统公平的上下文成本比较。

### EK-13 Agent-KB 以 “Student Guidance + Teacher Guidance” 双角色合成 BEGIN 提示
- **Layer**: L2 · **Type**: MEMORY · **Strength**: S3 · **Epistemic**: Fact · **Value**: B
- **Statement**: BEGIN 时把检索到的经验拆成“学生视角（怎么做）”与“教师视角（为什么/注意点）”两段合成一条 guidance。
- **Evidence**: E-024
- **Links**: `mechanism:EK-14`；`contrast:EK-23`（Dynamic-Cheatsheet 只给一份操作清单）
- **Note**: 这是“同一记忆、两种叙述视角”的少见实现。

---

## C. 写入与维护机制（EK-14 ~ EK-34）

### EK-14 Agent-KB 采用 append-only 四字段记录，但每次写入后全量重建索引
- **Layer**: L2 · **Type**: ENGINEERING · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: 成功任务被 LLM 摘要为 `agent_planning/search_agent_planning/agent_experience/search_agent_experience` 四字段后追加到 JSON；写入后重新构造 `AKB_Manager`，重建 TF-IDF 与 embedding 索引。
- **Evidence**: E-025, E-026
- **Links**: `causal:EK-33`（append 模式的代表）；`constraint:EK-09`（重建成本来自嵌入）；`contrast:EK-17`（Evolver 走 merge）
- **Note**: 写入成本 O(N)，随记忆量线性增长；这是 append-only 的直接代价。

### EK-15 Agent-Workflow 只从成功轨迹归纳 workflow，且仅在 BEGIN 提供
- **Layer**: L2 · **Type**: MEMORY · **Strength**: S3 · **Epistemic**: Fact · **Value**: B
- **Statement**: 成功轨迹被 LLM 诱导为可复用 workflow，embedding 检索；IN 阶段无返回。
- **Evidence**: E-027
- **Links**: `mechanism:EK-14`（同为“成功后抽象”）；`contrast:EK-24`（MobileE 提供 tips/shortcuts 双形态）
- **Note**: workflow 是“过程模板”，与 Agent-KB 的“经验片段”形成同项目内对照。

### EK-16 Dilu / Generative / Voyager 是 JSON + LLM 摘要的同构实现，且都只在 BEGIN 提供
- **Layer**: L1 · **Type**: MEMORY · **Strength**: S3 · **Epistemic**: Fact · **Value**: C
- **Statement**: 三者都以 JSON 持久化、用 LLM 生成摘要、BEGIN-only；差异主要在提示词与字段。
- **Evidence**: E-028, E-029, E-030
- **Links**: `mechanism:EK-14`；`subsystem:EK-07`；`contrast:EK-17`
- **Note**: 同构实现占据 13 个系统中的 3 个，说明“provider 数量”不等于“机制多样性”。

### EK-17 Evolver 用“原则库 + 成功率打分 + 语义合并”维护记忆
- **Layer**: L2 · **Type**: MEMORY · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: 记录含 `success_count/usage_count`；分数 `(success_count+1)/(usage_count+2)`；低于阈值剪枝；新记忆与旧记忆 cosine>0.8 且 LLM 判语义相同才合并，否则新建 uuid4。
- **Evidence**: E-031, E-032, E-033
- **Links**: `contrast:EK-14`（append vs merge）；`causal:EK-18`（分数更新依赖检索缓存）；`mechanism:EK-20`（MemP 的 revise 是第三种写法）
- **Note**: 这是 13 个系统里唯一显式“成功/使用”双计数的实现。

### EK-18 Evolver 的存盘/装载不对称：不存 embeddings，重启后全量重算
- **Layer**: L2 · **Type**: ENGINEERING · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: `_save_store` 只写 `{"memories": records}`，不写 `embeddings`；`initialize` 读 `data.get("embeddings", [])`，长度不符时对全部记录重新编码。
- **Evidence**: E-034
- **Links**: `constraint:EK-09`（384 维硬编码）；`causal:EK-17`；`contrast:EK-14`（Agent-KB 每次写入即重建，Evolver 每次启动重建）
- **Note**: 持久化设计遗漏导致启动成本随记忆量线性增长；但作者可能有意避免存储膨胀。

### EK-19 MemP 的记忆是“抽象脚本 + 具体步骤摘要”，并硬编码 top_k=1
- **Layer**: L2 · **Type**: MEMORY · **Strength**: S3 · **Epistemic**: Fact · **Value**: B
- **Statement**: `take_in_memory` 生成 <100 词 procedure script 与 <150 词 trajectory summary；`provide_memory` 只取 1 条。
- **Evidence**: E-036, E-020
- **Links**: `contrast:EK-17`（原则库 vs 脚本库）；`causal:EK-20`（失败时改写脚本）
- **Note**: 单条召回使“记忆干扰”最小，但也放弃多经验融合。

### EK-20 MemP 在任务失败时改写已有记忆，而不是新增
- **Layer**: L2 · **Type**: MEMORY · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: 失败时对已检索到的记忆执行 `_adjust_memory`：重写脚本/摘要、`revision_count += 1`、`status = revised_after_failure`；成功时才新增记录。
- **Evidence**: E-037
- **Links**: `mechanism:EK-33`（revise 模式的唯一实现）；`dependency:EK-05`（依赖 is_correct）；`contrast:EK-14`（append-only 不会修正旧记忆）
- **Note**: 这使记忆可以“被现实纠正”，但也可能把一次失败错误归因到本来正确的记忆。

### EK-21 MemP 在 is_correct 缺失时默认按成功处理
- **Layer**: L1 · **Type**: FAILURE · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: `raw_status = meta.get("is_correct", "true")`；缺失元数据被解释为成功，进而写入/强化记忆。
- **Evidence**: E-038
- **Links**: `causal:EK-20`（错误成功判定会触发 revise/新增的方向错误）；`contrast:EK-34`（其他 provider 缺失时默认 False）
- **Note**: 同一字段在不同 provider 中默认值相反（MemP/Evolver=true，Agent-KB=false）。

### EK-22 Expel 同时维护 insights 与成功轨迹两个库，并用 TF-IDF 检索
- **Layer**: L2 · **Type**: MEMORY · **Strength**: S3 · **Epistemic**: Fact · **Value**: B
- **Statement**: `insights.json` 存经验条目，`success_trajectories.json` 存成功轨迹；两者分别建 TF-IDF 矩阵，检索 top_k=1；失败/成功都抽 insight，成功另做 trajectory refine。
- **Evidence**: E-039, E-020
- **Links**: `mechanism:EK-16`（同为 JSON+摘要）；`contrast:EK-17`（无打分合并）
- **Note**: “经验”与“原始轨迹”双库并存，是记忆粒度的一种折中。

### EK-23 Dynamic-Cheatsheet 维护单一全局 cheatsheet，按相似 query 由 LLM 重写
- **Layer**: L2 · **Type**: MEMORY · **Strength**: S3 · **Epistemic**: Fact · **Value**: B
- **Statement**: 只保留一份 <200 词的 `global_cheatsheet.txt`；IN 阶段检索一条相似 query-trajectory，连同旧 cheatsheet 交给 curator LLM 合成新版本。
- **Evidence**: E-040
- **Links**: `mechanism:EK-33`（“整体重写”是第五种写入形态）；`contrast:EK-13`（双角色 vs 单份清单）；`constraint:EK-07`（仅 IN）
- **Note**: 全局单文件意味着后写覆盖先写，没有多记忆并存。

### EK-24 MobileE 用 tips 与 shortcuts 两套 JSON，检索是词法 token 匹配
- **Layer**: L2 · **Type**: MEMORY · **Strength**: S3 · **Epistemic**: Fact · **Value**: C
- **Statement**: tips 是自然语言建议，shortcuts 是键值型快捷操作；检索按 query token 是否出现在条目中匹配（非向量）。
- **Evidence**: E-052
- **Links**: `contrast:EK-09`（唯一不用 embedding 的实现之一）；`mechanism:EK-16`
- **Note**: 词法匹配在长 query/同义表达上召回弱，但零模型成本。

### EK-25 SkillWeaver 把“技能”落成 Python 文件并作为 API 型记忆返回
- **Layer**: L2 · **Type**: TOOLING · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: 生成技能写入 `storage/skillweaver/skillweaver_generated_skills.py`；加载为 registry，按名称匹配后经 `ToolWrapper` 包装成 callable；摄取要求 `is_correct` 与 `task_success` 同时为真。
- **Evidence**: E-051, E-004
- **Links**: `mechanism:EK-26`（Cerebra 也返回 API）；`constraint:EK-34`（双门控）；`contrast:EK-04`（TEXT vs API）
- **Note**: 这是“记忆即能力”的极端形态：记忆不是告诉 Agent 什么，而是给 Agent 新函数。

### EK-26 Cerebra 同时维护图记忆与工具记忆，响应可混合 TEXT 与 API
- **Layer**: L2 · **Type**: MEMORY · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: 图记忆由 nodes/edges 组成；工具记忆从轨迹抽取代码、安全检查后注册；`provide_memory` 可同时返回文本记忆与 `wrapped_tool`。
- **Evidence**: E-050, E-016, E-017
- **Links**: `mechanism:EK-25`；`causal:EK-27`；`subsystem:EK-28`
- **Note**: 1,601 行，是 13 个系统中复杂度最高的实现。

### EK-27 Cerebra 的检索是“混合打分 + 图邻居扩展 + 阶段阈值”
- **Layer**: L2 · **Type**: ENGINEERING · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: 先 text/semantic 加权打分，再沿语义边（阈值 0.75）扩展最多 3 个邻居；IN 阶段把 context 拼入 query 并追加 “continuation resume partial progress”，使用独立 `min_score_in_phase`。
- **Evidence**: E-016, E-017, E-018, E-019
- **Links**: `contrast:EK-10`；`causal:EK-07`（双阶段能力）；`mechanism:EK-28`
- **Note**: 这是唯一把“图结构”真正用于检索扩展的实现。

### EK-28 Cerebra 每 50 个任务做一次 consolidation
- **Layer**: L2 · **Type**: MEMORY · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: `consolidation_interval=50`；达到阈值后合并相似节点、剪边、优化权重，并持久化。
- **Evidence**: E-053
- **Links**: `mechanism:EK-33`（consolidate 模式）；`causal:EK-26`；`contrast:EK-32`（Lightweight 用容量触发剪枝）
- **Note**: 维护是“批量后台式”的，不是每次写入都做，写入成本被摊薄。

### EK-29 Lightweight 明确分短期/长期两层，但长期默认不注入
- **Layer**: L2 · **Type**: MEMORY · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: 短期记忆服务当前任务，长期记忆跨任务；`enable_longterm_provision` 默认 False，因此默认行为是“长期只写不读”。
- **Evidence**: E-041, E-042, E-043
- **Links**: `contrast:EK-07`（双阶段但长期关闭）；`causal:EK-30`；`causal:EK-31`
- **Note**: 这是最容易被误读为“双记忆都生效”的配置陷阱。

### EK-30 Lightweight 用 context 前缀差量抽取短期记忆，超限时 LLM 选择、失败回退 FIFO
- **Layer**: L2 · **Type**: ENGINEERING · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: 若当前 context 以上一轮为前缀，只取新增部分；否则取整段；delta<50 字符跳过。每若干步由 LLM 产出 `{step_summary, key_extracts≤3}`；短期超限时 LLM 选保留项，失败回退 FIFO。
- **Evidence**: E-044, E-045, E-046
- **Links**: `mechanism:EK-29`；`dependency:EK-08`（context 来自每步注入点）；`contrast:EK-28`（按任务数 vs 按步数维护）
- **Note**: 前缀差量是对“每步都传完整上下文”的成本优化，但假设上下文是单调追加的。

### EK-31 Lightweight 的长期记忆只写成功轨迹，并有数量/长度/去重/容量四重约束
- **Layer**: L2 · **Type**: MEMORY · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: 仅成功轨迹；每任务最多 2 strategic + 2 operational；长度<20 丢弃；SHA256 去重；超过 max+buffer 才剪枝，优先 LLM 选删除项，失败回退 `success_count*2+usage_count`。
- **Evidence**: E-047, E-048
- **Links**: `causal:EK-29`；`contrast:EK-17`（打分排序 vs LLM 选删）；`dependency:EK-05`
- **Note**: 这是 13 个系统中写入约束最完整的一套。

### EK-32 Lightweight 以 “strategic_N” 形式记录使用量，剪枝后索引可能位移
- **Layer**: L2 · **Type**: FAILURE · **Strength**: S3 · **Epistemic**: Observation（候选缺陷） · **Value**: A
- **Statement**: `used_memory_ids` 形如 `strategic_3`；更新使用量时按下划线解析索引；长期列表剪枝后元素前移，旧 id 可能指向另一条记忆。
- **Evidence**: E-049, E-048
- **Links**: `causal:EK-31`（剪枝导致位移）；`contrast:EK-17`（Evolver 用 uuid 标识）
- **Note**: 未运行验证，列为候选缺陷 C-01；若成立会污染分数统计。

### EK-33 五种植入/维护形态共存：append / merge / revise / consolidate / rewrite
- **Layer**: L2 · **Type**: MEMORY · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: Agent-KB 与 Agent-Workflow 是 append；Evolver 是 cosine+LLM 判定后 merge；MemP 是失败后 revise；Cerebra 是每 50 任务 consolidate；Dynamic-Cheatsheet 是整体 rewrite。
- **Evidence**: E-014, E-017, E-020, E-023, E-028
- **Links**: `mechanism:EK-01`（都由同一接口承载）；`contrast:EK-34`；`causal:EK-19`
- **Note**: 这是本轮最重要的横向对照结论：**“记忆系统”的差异主要是写入形态，而不是存储介质**。

### EK-34 “成功才写入”的门控在 13 个系统中至少有 5 种写法，且缺失默认值不一致
- **Layer**: L2 · **Type**: FAILURE · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: Agent-KB 缺失即 False；MemP/Evolver 缺失即 True；SkillWeaver 要求 is_correct 且 task_success；Expel 失败也抽 insight；Cerebra/Lightweight 走各自成功判定。`_is_task_correct` 在 analyzer/trajectory_tools/provider 中重复实现。
- **Evidence**: E-025, E-058, E-059, E-038, E-051, E-039
- **Links**: `causal:EK-33`；`dependency:EK-05`；`contrast:EK-21`
- **Note**: 同一个 `TrajectoryData` 在不同 provider 中会得到相反的处理结果。

---

## D. 元进化管线（EK-35 ~ EK-45）

### EK-35 元进化是四阶段状态机：Analyze→Generate→Create→Validate，状态落 `state.json`
- **Layer**: L1 · **Type**: WORKFLOW · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: `MemoryEvolver` 按 `phases.{analyze,generate,create,validate}.completed` 顺序执行，支持从中断处续跑。
- **Evidence**: E-054, E-055
- **Links**: `causal:EK-36`；`causal:EK-39`；`causal:EK-43`
- **Note**: 阶段之间通过文件/目录传递产物，而不是内存对象。

### EK-36 分析阶段是“工具型 Agent 读轨迹”，且自身不被记忆增强
- **Layer**: L2 · **Type**: AGENT · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: `AnalysisAgent` 挂载 TrajectoryViewer（最多 3 任务）/StepViewer/MemoryDatabaseViewer，`max_steps=20`；`ToolCallingAgent(memory_provider=None)`。
- **Evidence**: E-056, E-057
- **Links**: `causal:EK-35`；`contrast:EK-08`（执行 Agent 有记忆，分析 Agent 没有）
- **Note**: 分析者不被被分析对象影响，是一种朴素的“观察者隔离”。

### EK-37 生成 prompt 把 `memory_types.py` 与目标 provider 源码全文注入，温度随 creativity 线性放大
- **Layer**: L2 · **Type**: ENGINEERING · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: prompt 含指定 provider 源码、`memory_types.py` 全文；“温度 = 0.3 + 0.9*creativity”，`max_tokens=60000`；原始输出落 `llm_raw_output_<ts>.txt`。
- **Evidence**: E-060, E-061
- **Links**: `causal:EK-38`；`constraint:EK-03`（必须理解注册表格式）
- **Note**: 高 max_tokens + 全文注入使单次生成成本很高。

### EK-38 生成结果的解析依赖正则与固定标记，格式偏差会直接丢字段
- **Layer**: L2 · **Type**: FAILURE · **Strength**: S3 · **Epistemic**: Fact · **Value**: B
- **Statement**: 解析 `**Class Name**` / `**Module Name**` / `**Enum Name**` / `**Enum Value**` / ```python 代码块 / `**key**: value`；无 schema 校验，缺字段时下游才失败。
- **Evidence**: E-062
- **Links**: `causal:EK-37`；`causal:EK-39`（creator 假定字段齐全）
- **Note**: 生成→解析之间没有契约层，属于“字符串协议”。

### EK-39 Creator 的写入顺序是 provider → memory_types → config，依赖注释锚点插入
- **Layer**: L2 · **Type**: ENGINEERING · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: 先写 `EvolveLab/providers/<module>.py`（存在即失败），再用锚点 `add new memory type upside this line(Enum)` / `(PROVIDER_MAPPING)` 插入 enum 与 mapping，最后按锚点插 config 块。
- **Evidence**: E-063
- **Links**: `causal:EK-40`；`dependency:EK-03`；`constraint:EK-42`
- **Note**: 代码生成不是通过 AST 修改，而是字符串锚点替换。

### EK-40 Creator 的写入是非原子的：前两步成功、第三步失败时无回滚
- **Layer**: L2 · **Type**: FAILURE · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: provider 文件与 enum/mapping 先落盘；若 config_updates 校验失败，已写入的文件和注册表项不会回滚，留下“注册了但配置缺失”的半成品。
- **Evidence**: E-064, E-063
- **Links**: `causal:EK-39`；`contrast:EK-35`（阶段状态机有 checkpoint，但代码变异没有事务）；`causal:EK-41`
- **Note**: 这是元进化系统最危险的工程缺陷之一：搜索空间会被自己的失败样本污染。

### EK-41 config_updates 校验失败只告警，不阻断进化流程
- **Layer**: L2 · **Type**: FAILURE · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: `MemoryEvolver.generate` 对 `config_updates` 调 `_validate_config_updates`，捕获异常后仅打印 warning，注释写明 “allow user to proceed”。
- **Evidence**: E-065
- **Links**: `causal:EK-40`；`constraint:EK-39`；`contrast:EK-43`（validator 反而强制）
- **Note**: “生成阶段宽容、验证阶段强制”形成不对称。

### EK-42 静态检查是子串存在性检查，不是语义/可运行检查
- **Layer**: L2 · **Type**: VALIDATION · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: `ast.parse` + 检查源码是否包含 `BaseMemoryProvider`、`def provide_memory`、`def take_in_memory`、`def initialize` 及 import 文本；不检查签名、返回值、异常路径。
- **Evidence**: E-066
- **Links**: `dependency:EK-43`（真正的行为检查靠快检）；`constraint:EK-03`
- **Note**: 只要注释里出现这些字符串也可能通过静态检查。

### EK-43 验证器文档称“不自动修复”，实现默认开启自动修复
- **Layer**: L2 · **Type**: FAILURE · **Strength**: S3 · **Epistemic**: Fact（文档-实现矛盾） · **Value**: A
- **Statement**: `phase_validator.py` docstring 写 “static checks ... without automatic fixes”，但构造函数默认 `enable_auto_fix=True`、`max_fix_attempts=3`；`MemoryEvolver` 调 `PhaseValidator` 时未覆盖该参数。
- **Evidence**: E-067, E-068
- **Links**: `causal:EK-44`；`causal:EK-45`；`contrast:EK-42`（静态检查 vs 自动修复）
- **Note**: 这是“文档承诺 ≠ 实现事实”的典型样本。

### EK-44 快检在 EvolveLab+storage 的隔离副本中执行，并清理模块缓存
- **Layer**: L2 · **Type**: VALIDATION · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: 复制 `EvolveLab` 与 `storage` 到隔离目录；`del sys.modules['EvolveLab.*']`；按 `PROVIDER_MAPPING` 导入；注入 model；要求 `initialize()==True`、BEGIN 返回类型正确、`take_in_memory` 成功并观察到 storage 变化；出现 `[ERROR]` 日志判失败。
- **Evidence**: E-069, E-070
- **Links**: `contrast:EK-42`；`causal:EK-43`；`constraint:EK-03`
- **Note**: 隔离是“同进程目录隔离”，不是容器/进程隔离。

### EK-45 自动修复会把修复后的代码用 copy2 覆盖真实 provider 文件
- **Layer**: L2 · **Type**: GOVERNANCE · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: 隔离副本验证失败 → `SWEAgentValidator.fix_memory_system` 修复副本 → 若通过，`_sync_to_real_environment` 以时间戳 `.backup_` + `copy2` 覆盖真实 `EvolveLab/providers/<name>_provider.py`。
- **Evidence**: E-068
- **Links**: `causal:EK-43`；`contrast:EK-40`（代码变异无事务，但修复有备份）；`dependency:EK-44`
- **Note**: 备份只有单份时间戳文件，无恢复流程/审计日志。

---

## E. 搜索、选择与运行（EK-46 ~ EK-50）

### EK-46 每轮开始重置 provider storage 以保证“同一起点”
- **Layer**: L2 · **Type**: WORKFLOW · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: Step0 把 `./storage/{current_provider}` 重命名为时间戳备份，provider 随后从空存储开始，注释标注 “Fairness”。
- **Evidence**: E-071
- **Links**: `causal:EK-47`；`contrast:EK-31`（长期记忆被跨轮丢弃）；`dependency:EK-29`
- **Note**: 公平性以“遗忘上一轮全部记忆”为代价；备份目录没有清理策略。

### EK-47 锦标赛流程：基座 x 任务 → 生成 m 系统 → 同批评估 → top-t → finalists 复评
- **Layer**: L2 · **Type**: WORKFLOW · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: x=20 基座；m=3 候选；基座与 validated 系统在同一 batch 并行评估；top-t（默认 2）；finalists 在 `random.sample(batch,y=5)` + 新 x 任务上复评；winner 写 state；`dataset_cursor += 2x`。
- **Evidence**: E-072, E-073, E-074, E-076
- **Links**: `causal:EK-48`；`dependency:EK-49`；`contrast:EK-46`
- **Note**: 选择集与最终评估集有重叠（finalists 复评含原 batch 的 y 样本）。

### EK-48 默认选择是 accuracy 降序 + tokens 升序的字典序，Pareto 是可选分支
- **Layer**: L2 · **Type**: DECISION · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: `use_pareto_selection=False` 默认；默认 `_select_top` 按 accuracy 降序、tokens 升序；Pareto 权重 0.6 accuracy / 0.25 token / 0.15 time。
- **Evidence**: E-075
- **Links**: `causal:EK-47`；`contrast:EK-17`（成功率/使用率打分 vs 纯准确率排序）；`constraint:EK-46`
- **Note**: 字典序意味着 accuracy 相同才比较 token，容易把“更贵但略准”的系统排在“便宜但同准”之前——需核对实现细节。

### EK-49 Runner 以无超时子进程调用，固定 concurrency=1，任务索引转 1-based
- **Layer**: L2 · **Type**: RUNTIME · **Strength**: S3 · **Epistemic**: Fact · **Value**: B
- **Statement**: `Popen(...).wait()` 无 timeout；`--concurrency 1`；`task_indices` 0-based 传入后 +1 转 CLI 参数；非 0 退出抛 RuntimeError。
- **Evidence**: E-078, E-079
- **Links**: `causal:EK-47`（评估吞吐受限）；`contrast:EK-50`（无测试兜底）
- **Note**: 单任务卡死会使整轮进化挂起；没有 per-task 超时或重试。

### EK-50 核心逻辑无自动化测试，指标聚合是简单均值
- **Layer**: L2 · **Type**: TESTING · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: `EvolveLab/`、`MemEvolve/` 无测试文件；测试仅在 vendored `mini-swe-agent`。`trajectory_tools` 聚合 accuracy 为 1/0 均值、tokens 为均值。
- **Evidence**: E-091, E-083
- **Links**: `constraint:EK-49`（无测试 → 只能靠运行时验证）；`contrast:EK-44`（快检是唯一门）
- **Note**: 没有方差/置信区间/显著性检验，锦标赛结果的可重复性未知。
