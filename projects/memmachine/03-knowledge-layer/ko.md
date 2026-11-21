# Knowledge Layer — KO-01 ~ KO-08

> 升维纪律：每个 KO 只能从 EK 图上的簇/链生成，必须声明 `aggregation_rule`（R1-R4）、簇内 EK、解释范围扩大论证、≥3 条定向反例、scope 与回溯链。
> **证据纪律**：本轮只有 S3（静态代码）与 S2（设计文档），没有执行测试/服务。因此所有 KO 的 `Epistemic` 标为 **Observation（single-project, code-level）**，不写 `Validated Pattern`、不写 `Principle`、不写 Law；跨项目验证一律 pending。
> **层級**：这些是 L3 candidate patterns；若独立验证判定过度升维，允许降为 L2 或 Hypothesis。

---

## KO-01 记忆系统不是“存一份”，而是“权威原文 + 可重建索引”的二元契约
- **Layer**: L3 candidate · **Epistemic**: Observation（single-project；cross-project pending） · **Value**: A
- **aggregation_rule**: **R1 机制簇** — EK-03（declarative 原文+派生+边）、EK-04（event segment+derivative+episode store）、EK-18（节点→边）、EK-20（五阶段）、EK-21（segment 先于 vector）、EK-32（derivative 召回→episode 回填）、EK-33（segment 召回→窗口）、EK-34（单位不等价）、EK-51（漂移告警）共享同一机制：**原文是权威，索引是可重建视图，检索必须能回填原文**。
- **簇内 EK**: 9 条；边类型：mechanism 3 + dependency 3 + contrast 2 + causal 1。
- **解释范围扩大论证**:
  从“declarative 这样写、event 那样写”提升为“**任何多表示记忆系统都必须回答：哪一份是权威、哪一份可重建、删除时先断哪一侧**”。这解释了为什么两个 backend 的写/删顺序相反却都成立：写入时先保权威（segment 先于 vector），删除时先断检索（vector 先于 segment）。它也能解释为什么“索引漂移”只会导致少结果、不会返回错原文（EK-51）。
- **反例（定向搜索 3 条）**:
  1. **反例 A（event 返回单位不是原文）**：event 检索返回的是 segment 窗口，不是完整 episode；`episode_storage` 只用于水合，窗口仍可能截断对话。所谓“回填原文”在 event 路径是“回填 episode + 保留 segment 边界”，不是无损还原（EV-031）。
  2. **反例 B（边失败使契约失效）**：declarative 先写节点、后写 `DERIVED_FROM` 边；边失败时 derivative 可被搜到但无法回填 episode。契约是设计意图，不是运行时保证（EV-026）。
  3. **反例 C（event 的权威可能是 segment store，不是 episode store）**：event 写入五阶段里，episode 已在顶层先写，但 event 的 segment/derivative 关系先于向量落库；若把“权威”定义为“可重建索引的最小集合”，event 的权威其实是 segment store，而不是 episode（EV-030、EV-032）。
- **Scope**: 仅适用于 MemMachine 的 declarative/event 两条 LTM 路径，以及其上的检索/删除；不适用于纯 append-only 日志或无索引系统。
- **回溯**: KO-01 → EK-03/04/18/20/21/32/33/34/51 → EV-026/030/031/032/033/028

---

## KO-02 多存储写入是一个没有补偿的 saga：失败不会崩溃，只会让记忆悄悄变少
- **Layer**: L3 candidate · **Epistemic**: Observation（single-project；cross-project pending） · **Value**: A
- **aggregation_rule**: **R2 因果链簇** — EK-14（顶层条件多路并行）→ EK-15（return_exceptions 聚合）→ EK-16（TODO 无 write intent）→ EK-17（内层 fail-fast）→ EK-18/EK-21（backend 内部分步）→ EK-51（读侧漂移）。
- **簇内 EK**: 8 条；边类型：causal 5 + contrast 2 + dependency 1。
- **解释范围扩大论证**:
  从“某处 `gather` 没回滚”提升为“**当一份记忆被复制到 N 个存储，而系统没有 write-intent/补偿日志时，部分失败不会表现为错误，而会表现为‘某些记忆以后再也查不到’**”。这条链可以解释三个看似无关的现象：event 索引漂移 warning、semantic consolidation 丢记忆、session 删除残留；它们的共同结构是“先写/先删一部分，后续步骤失败，系统继续运行”。
- **反例（定向搜索 3 条）**:
  1. **反例 A（重试可能造成重复而不是丢失）**：调用方收到统一抛错后若重试整批，已成功的 episode 可能被再写一次；没有幂等键的路径会产生重复，而不是“变少”。所以“失败 → 变少”不是唯一后果（EV-016；`EpisodeEntry.uid` 是否客户端可控需另证）。
  2. **反例 B（删除顺序降低坏读）**：event 删除先删 vector 再删 segment，最坏情况是“原文还在但搜不到”，而不是“搜到但原文缺失”。这把失败方向从“坏读”转成“少读”，是设计上的缓解（EV-032）。
  3. **反例 C（并非所有写入都同等关键）**：semantic 是异步派生；semantic 失败时原始 episode 仍在，记忆并未整体丢失。把“三路写”一概称为 saga 会高估 semantic 路径的严重性（EV-045、EV-046）。
- **Scope**: 适用于“同一逻辑记忆写入 ≥2 个独立存储/索引”的路径；不适用于单存储或强事务系统。
- **回溯**: KO-02 → EK-14/15/16/17/18/21/28/51 → EV-016/017/019/026/030/033/050/051

---

## KO-03 记忆能力由“配置完整性”决定：缺一项就整层关闭，而不是降级运行
- **Layer**: L3 candidate · **Epistemic**: Observation（single-project；cross-project pending） · **Value**: A
- **aggregation_rule**: **R2 因果链簇** — EK-06（配置树）→ EK-07（默认关闭）→ EK-08（LTM 全有或全无）→ EK-09（凭证缺失自动关闭）→ EK-02（backend 参数必填）。
- **簇内 EK**: 5 条；边类型：causal 3 + constraint 2。
- **解释范围扩大论证**:
  从“默认 `enabled=false`”提升为“**在 memory-as-a-service 中，‘是否具备记忆’不是单一开关，而是由后端参数、模型凭证、资源 ID 共同决定的合取条件；任一不满足，系统选择整层关闭而不是部分可用**”。这解释了为什么部署成功、健康检查通过，仍然可能完全没有长期记忆；也解释了为什么不同 backend 的“最小可用配置”不同。
- **反例（定向搜索 3 条）**:
  1. **反例 A（部分能力仍可存活）**：LTM 被关闭不必然关闭 STM；STM 只需要 `llm_model`，可在 LTM 缺失时继续提供短期摘要。因此“整层关闭”只适用于 LTM，不是整个 episodic（EV-020、EV-022）。
  2. **反例 B（非 OpenAI 路径不受凭证门控影响）**：自动关闭只针对 OpenAI `api_key` 为空且无 `base_url`；Bedrock、SentenceTransformers、Cohere 等 provider 不走这条判断。把“缺凭证 → 关闭”泛化到所有 provider 是过度概括（EV-024）。
  3. **反例 C（运行时可重新启用）**：`/config` 可以在运行期改配置/加资源；关闭不是永久的，只要配置 API 开启且资源可用即可恢复（EV-075）。因此“缺一项就关闭”是启动时判定，不是不可逆状态。
- **Scope**: 适用于启动时的 `Configuration` + `MemMachine` 初始化；运行期变更的生效边界仍需实测（C-04）。
- **回溯**: KO-03 → EK-02/06/07/08/09 → EV-020/021/023/024/070/072

---

## KO-04 上下文单位是检索语义的一等参数：同一 `expand_context` 在两个 backend 不等价
- **Layer**: L3 candidate · **Epistemic**: Observation（single-project；cross-project pending） · **Value**: A
- **aggregation_rule**: **R4 主题簇** — EK-19（派生粒度）、EK-32（episode 邻域）、EK-33（segment 窗口）、EK-34（单位不等价）、EK-35（reranker 换分）、EK-42（top_k/expand 默认）、EK-43（query 重排）。
- **簇内 EK**: 7 条；边类型：contrast 3 + dependency 2 + constraint 1 + causal 1。
- **解释范围扩大论证**:
  从“两个实现不同”提升为“**在记忆检索中，‘返回多少上下文’不是数量参数，而是单位参数；单位不同时，top_k、阈值、分数都不可直接比较**”。这解释了为什么同一 `top_k=10` 在两个 backend 下可能返回 10 个 episode 邻域或 10 个 segment 窗口；也解释了为什么 reranker 存在时分数语义变化（EK-35），以及为什么 query 重排会改变 LLM 看到的顺序（EK-43）。
- **反例（定向搜索 3 条）**:
  1. **反例 A（`expand_context=0` 也不是同一单位）**：declarative 的 0 表示“只返回命中 episode”；event 的 0 仍以 segment 为单位返回命中窗口，测试 `test_expand_context_zero_returns_matches_in_score_order` 只验证顺序，不验证单位一致（EV-031、EV-060~062 的测试文件）。
  2. **反例 B（reranker 让分数不可比）**：启用 reranker 后，event 用 reranker 分数、declarative 对整段 context 打分；“分数高低”不是同一模型产出，跨 backend 阈值比较无意义（EV-028、EV-031、EV-035）。
  3. **反例 C（top_k 作用对象不同）**：declarative 先对 derivative 取 `limit=min(5*max,200)` 再回填 episode；event 对 segment 取 top_k；API 的 `top_k` 最终约束的是“返回条目数”，不是“候选数”。三者不是同一层（EV-028、EV-031、EV-068）。
- **Scope**: 适用于 MemMachine 的 LTM 检索与 API 参数语义；不适用于只返回固定记录单位的系统。
- **回溯**: KO-04 → EK-19/32/33/34/35/42/43 → EV-027/028/031/055/067/068

---

## KO-05 身份是命名空间约定，不是信任边界
- **Layer**: L3 candidate · **Epistemic**: Observation（single-project；cross-project pending） · **Value**: A
- **aggregation_rule**: **R3 不变量簇** — EK-05（session_key）、EK-10（REST 无认证）、EK-11（MCP header）、EK-43（API schema）、EK-48（session 删除）共同维持一个不变量：**服务端只按调用方声明的 org/project 分区，不验证声明是否可信**。
- **簇内 EK**: 5 条；边类型：constraint 2 + mechanism 1 + subsystem 2。
- **解释范围扩大论证**:
  从“router 没写 auth”提升为“**多租户记忆服务的隔离强度 = 命名空间约定强度 × 前置网关认证强度**”。这解释了为什么 REST 请求体、MCP header、session key、semantic set 都使用同一组字符串，却没有一个地方做签名/令牌校验；也解释了为什么 `org_id` 只做字符校验（拒绝 `/`）而不做归属校验。
- **反例（定向搜索 3 条）**:
  1. **反例 A（网关可补足）**：部署在带 OAuth/API-key 的网关后，服务端的“无认证”不构成越权；网关可以把身份注入为 org/project。此时不变量仍成立，但风险被外部化（EV-012、EV-013）。
  2. **反例 B（单租户部署无意义）**：单租户/本地 stdio 场景下，org/project 只是常量命名空间，伪造没有收益；不能用多租户风险反推所有部署都不安全。
  3. **反例 C（字符校验不是归属校验）**：`org_id` 含 `/` 被拒只能防止 key 结构破坏，不能防止调用方声明他人的 org；把它当成“租户校验”是误读（EV-069、EV-083）。
- **Scope**: 适用于 REST/MCP 直连部署；不适用于由可信网关强制注入身份且服务不可直连的部署。
- **回溯**: KO-05 → EK-05/10/11/43/48 → EV-012/013/015/069/083

---

## KO-06 记忆写入是两阶段的：原始 episode 同步落库，结构化 profile 异步派生
- **Layer**: L3 candidate · **Epistemic**: Observation（single-project；cross-project pending） · **Value**: A
- **aggregation_rule**: **R2 因果链簇** — EK-22（semantic queue）→ EK-23（5 条/批）→ EK-24（50 feature 上下文）→ EK-25（ADD/DELETE）→ EK-26（citation）→ EK-27（20 条触发 consolidation）。
- **簇内 EK**: 6 条；边类型：causal 4 + constraint 1 + contrast 1。
- **解释范围扩大论证**:
  从“semantic 是异步的”提升为“**同一份对话在系统里有两个不同时效的版本：可立即检索的原始 episode，与稍后才出现的结构化 feature；两者的‘写入成功’语义不同**”。这解释了为什么 `add_messages` 返回成功但 `search` 暂时查不到 profile；也解释了为什么 semantic 的失败不会丢失原文，却会丢失“可聚合的知识”。
- **反例（定向搜索 3 条）**:
  1. **反例 A（semantic 可关闭）**：`semantic_memory.enabled=False` 时只有第一阶段；此时不存在两阶段，不能把“所有写入”都描述为两阶段（EV-023）。
  2. **反例 B（ingestion 可能很快追上）**：后台 worker 若在 `add_message` 返回后立即处理，用户观察到的时间差可能接近 0；两阶段是结构性的，不保证可观测延迟（EV-046）。
  3. **反例 C（原文也可能异步）**：episode store 写是 `await`，但 STM/LTM 的具体落库可能排队；把“原文一定同步”当成绝对保证会高估一致性（EV-016、EV-019）。
- **Scope**: 适用于启用 semantic memory 的部署；不适用于纯 episodic 模式。
- **回溯**: KO-06 → EK-22/23/24/25/26/27 → EV-023/045/046/047/048/049

---

## KO-07 破坏性 consolidation + provenance 膨胀：合并越彻底，追溯越不可信
- **Layer**: L3 candidate · **Epistemic**: Observation（single-project；cross-project pending） · **Value**: A
- **aggregation_rule**: **R2 因果链簇** — EK-27（阈值触发）→ EK-28（先删后加）→ EK-29（citation 合并到每个新 feature），并由 EK-25/26（ADD/DELETE 与 citation 原语）提供机制前提。
- **簇内 EK**: 5 条；边类型：causal 4 + mechanism 1 + contrast 1。
- **解释范围扩大论证**:
  从“合并实现有 bug”提升为“**当‘去重’同时承担‘删除冗余’和‘保留证据’两个目标时，删除会破坏证据的粒度，而 citation 合并会把破坏伪装成完整**”。这解释了为什么 consolidation 后 feature 数量下降、但每个 feature 的 citation 数量上升；也解释了为什么“有 citation”不等于“可定位到支持它的原句”。
- **反例（定向搜索 3 条）**:
  1. **反例 A（可能不删）**：若 LLM 返回 `keep_memories` 覆盖全部旧 feature，`memories_to_delete` 为空，则没有破坏；破坏性只在“确有删除”时发生（EV-050）。
  2. **反例 B（context 超限会中止）**：`_ConsolidationContextLengthError` 在删除前抛出，整次 consolidation 中止；这保护了旧 feature，但不是事务回滚（EV-049、EV-050）。
  3. **反例 C（citation 合并不必然错误）**：如果多个旧 feature 确实由同一组消息支持，合并 citation 是合理的去重；问题在于代码把**所有**旧 citation 挂到**每个**新 feature，而不做“新 feature ↔ 子集”的映射（EV-052）。
- **Scope**: 适用于 semantic consolidation（按 tag 分组、阈值 20）；不适用于纯 append-only 记忆。
- **回溯**: KO-07 → EK-25/26/27/28/29 → EV-048/049/050/051/052

---

## KO-08 生命周期是“引用计数 + 延迟清理 + 最终一致”，不是“删除即释放”
- **Layer**: L3 candidate · **Epistemic**: Observation（single-project；cross-project pending） · **Value**: A
- **aggregation_rule**: **R4 主题簇** — EK-31（删除队列）、EK-45（STM 驱逐）、EK-47（实例 LRU/refcount）、EK-48（逻辑删除）、EK-49（TaskGroup）、EK-50（共享 segment registry）共同覆盖“内存实例、短期记忆、会话、segment 分区”四层生命周期。
- **簇内 EK**: 7 条；边类型：causal 3 + mechanism 2 + dependency 1 + subsystem 2。
- **解释范围扩大论证**:
  从“删除要等一会儿”提升为“**长生命周期对象的安全释放必须同时满足三件事：引用计数防悬空、延迟队列解耦关闭成本、持久化状态机支持重试**”。这解释了为什么 `erase` 在 ref_count>0 时抛错、为什么 session 删除先改 status 再入队、为什么 segment store 用 registry + purge queue 而不是 `DROP`。
- **反例（定向搜索 3 条）**:
  1. **反例 A（capacity=0 时不缓存）**：样例 `instance_cache_size=0` 表示用完即入 disposal queue，不存在长期驻留；refcount 机制主要保护“正在使用”的短窗口，而非长期缓存（EV-049、EV-073）。
  2. **反例 B（lease 防重复，不防失败）**：SQL lease 只保证一个 worker 处理同一 session，不能保证 TaskGroup 内每个存储删除成功；失败后依赖下一次扫描，而不是即时修复（EV-043）。
  3. **反例 C（物理删除可能立即完成）**：若队列为空且 worker 空闲，删除可能在返回后很快完成；最终一致是上限，不是固定延迟（EV-042、EV-043）。
- **Scope**: 适用于进程内实例、STM、session、event segment store；不适用于无状态服务。
- **回溯**: KO-08 → EK-31/45/47/48/49/50/52 → EV-034/042/043/049/074/092

---

## 反例预算检查

| KO | 定向反例数 | 是否给出“为什么不是普遍规律” | 结论 |
|---|---:|---|---|
| KO-01 | 3 | 是（event 单位、边失败、权威定义） | 保留 L3 candidate |
| KO-02 | 3 | 是（重试重复、删除方向、semantic 非关键） | 保留 L3 candidate |
| KO-03 | 3 | 是（STM 可存活、provider 差异、运行时可恢复） | 保留 L3 candidate |
| KO-04 | 3 | 是（0 也不同单位、reranker 分数、top_k 层级） | 保留 L3 candidate |
| KO-05 | 3 | 是（网关、单租户、字符校验） | 保留 L3 candidate |
| KO-06 | 3 | 是（semantic 可关、追上、原文也可能排队） | 保留 L3 candidate |
| KO-07 | 3 | 是（可能不删、超限中止、citation 可合理合并） | 保留 L3 candidate |
| KO-08 | 3 | 是（capacity=0、lease 不防失败、可立即完成） | 保留 L3 candidate |

## KO → EK / Evidence 回溯索引

| KO | EK | EV |
|---|---|---|
| KO-01 | EK-03, 04, 18, 20, 21, 32, 33, 34, 51 | EV-026, 028, 030, 031, 032, 033 |
| KO-02 | EK-14, 15, 16, 17, 18, 21, 28, 51 | EV-016, 017, 019, 026, 030, 033, 050, 051 |
| KO-03 | EK-02, 06, 07, 08, 09 | EV-020, 021, 023, 024, 070, 072 |
| KO-04 | EK-19, 32, 33, 34, 35, 42, 43 | EV-027, 028, 031, 055, 067, 068 |
| KO-05 | EK-05, 10, 11, 43, 48 | EV-012, 013, 015, 069, 083 |
| KO-06 | EK-22, 23, 24, 25, 26, 27 | EV-023, 045, 046, 047, 048, 049 |
| KO-07 | EK-25, 26, 27, 28, 29 | EV-048, 049, 050, 051, 052 |
| KO-08 | EK-31, 45, 47, 48, 49, 50, 52 | EV-034, 042, 043, 049, 074, 092 |
