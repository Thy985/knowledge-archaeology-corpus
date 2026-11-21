# 02 · Engineering Knowledge（EK Graph · 36 条）

> 每条 EK = 一条**可检验的工程解释**（L1 实现 / L2 经验），不是模块说明。
> 质量纪律：平均出边 ≥1、游离 EK <20%、每条 ≥1 link。
> Epistemic：`Fact`（代码直接可见）/ `Observation`（多事实归纳但未跨项目）/ `Hypothesis`（推断）。

---

## A. 存储与一致性（EK-01 ~ EK-07）

### EK-01 内容与索引是「源—派生」双存储，而不是双写对等
- **claim**：`viking://` 内容存 RAGFS/AGFS；向量库只存可重建的检索层。内容写入是权威的，索引失败不改变源。
- **机制**：item 的 `abstract` 来自向量 payload；正文在文件；`rm` 先删索引后删文件（EV-36）；向量化在文件写成功之后（EV-33）。
- **evidence**：EV-03 / EV-33 / EV-36 / EV-40
- **links**：`causal→EK-02`（源优先决定了删除顺序）、`causal→EK-18`（写后才向量化）、`contrast→EK-05`（资源导入则先内容后入队，二者同向）、`mechanism→EK-13`
- **layer**：L2 · **value**：A · **epistemic**：Observation

### EK-02 删除采用「先索引、后文件」的失败安全方向
- **claim**：`rm` 先 `vector_store.delete_uris`，再 `_async_agfs.rm`。若中途崩溃，最坏结果是**文件残留 + 索引缺失**（漏检），而不是**文件已删但索引仍指向**（坏结果/幽灵命中）。
- **机制**：文档口号 "Better to miss a search result than to return a bad one"；代码顺序与文档一致（EV-36）。
- **evidence**：EV-36 / EV-38 / `09-transaction.md`
- **links**：`causal→EK-03`（顺序带来"孤儿索引"问题，需幂等补偿）、`constraint→EK-01`、`mechanism→EK-04`（mv 同样先改索引再删源）、`contrast→EK-18`（写路径是"源先、索引后"，方向相反但同一不变量）
- **layer**：L2 · **value**：A · **epistemic**：Fact

### EK-03 删除对「目标已不存在」做幂等补偿
- **claim**：目标不存在时不直接报错，而走 `_finish_missing_delete` 清理孤儿索引；`rm` docstring 明写 idempotent。
- **机制**：先 stat 判存在与类型；missing 分支只清索引、不删文件。
- **evidence**：EV-38
- **links**：`dependency→EK-02`（为 EK-02 的漏检窗口提供补偿）、`mechanism→EK-04`、`subsystem→EK-05`
- **layer**：L1 · **value**：B · **epistemic**：Fact

### EK-04 移动是「复制 → 改索引 → 删源」，而不是 rename
- **claim**：`mv` 不依赖底层原子 rename，而是显式三段式；崩溃时可出现源与目标并存，靠后续清理/重试收敛。
- **机制**：取源/目标双锁；目录复制后删除副本中的锁文件；VectorDB URI 更新失败则清目标；最后删源。
- **evidence**：EV-39 / `09-transaction.md`
- **links**：`mechanism→EK-02`（同一"不留幽灵索引"取向）、`dependency→EK-06`（双锁保证）、`contrast→EK-03`（rename 则不需要补偿）
- **layer**：L2 · **value**：B · **epistemic**：Observation（文档）

### EK-05 资源导入把「内容落地」放在锁内、把「派生计算」推到队列
- **claim**：`add_resource` 的同步段只做 R/N/F/V 快照 + `ContextUpdatePlan` + 内容写；index/semantic 都入队。用户拿到成功响应时，检索层可能尚未就绪。
- **机制**：final_uri 资源锁 → 快照 → 编译 plan → 同步内容 → 清临时产物 → 入队 → 移交锁。
- **evidence**：EV-40 / EV-41
- **links**：`causal→EK-10`（异步派生造成"已导入但检索不到"窗口）、`constraint→EK-06`（锁覆盖同步段）、`mechanism→EK-01`
- **layer**：L2 · **value**：A · **epistemic**：Observation（文档）

### EK-06 锁有三种粒度语义，且 Tree 不自动扩展到父目录
- **claim**：Exact 只保护精确路径；Tree 保护子树但不阻止父目录名被独立操作；`Tree(/docs/a.md)` 不会退化为 `Tree(/docs)`。
- **机制**：锁类型 E/T/MV；路径必须绝对，拒绝 `..`、NUL、重复斜杠；目录操作取 tree，文件取 exact。
- **evidence**：EV-37 / EV-42 / EV-44 / EV-45
- **links**：`constraint→EK-05`（导入用 final_uri 资源锁）、`constraint→EK-04`（mv 双锁）、`dependency→EK-07`（协议版本决定语义）
- **layer**：L2 · **value**：A · **epistemic**：Observation（文档；Rust-unread）

### EK-07 锁协议升级不能滚动混跑，必须停写→排空→换代
- **claim**：新旧 token 格式/字段不同，混跑会让锁互不可见；文档要求停写、排空、切换、确认旧 token 释放。
- **机制**：token `{owner_id}:{time_ns}:{lock_type}`；旧 `.path.ovlock` 仅兼容读。
- **evidence**：EV-42 / EV-43
- **links**：`constraint→EK-06`、`causal→EK-04`（锁失效会让 mv 并发复制）
- **layer**：L2 · **value**：B · **epistemic**：Observation（文档）

---

## B. 会话与生命周期（EK-08 ~ EK-10）

### EK-08 会话提交是两阶段，但只有 Phase 1 是"锁内原子发布"
- **claim**：Phase 1 持 session root EXACT 锁完成 archive 切分、持久化、任务入队、retained 更新与 marker；Phase 2 完全在锁外做模型密集型工作。
- **机制**：返回值含 `task_id`；无可归档消息返回 skipped；Phase 2 失败不阻塞客户端。
- **evidence**：EV-16 / EV-17
- **links**：`causal→EK-09`、`constraint→EK-10`、`mechanism→EK-06`
- **layer**：L2 · **value**：A · **epistemic**：Observation（文档）

### EK-09 恢复靠持久队列，不靠 redo log
- **claim**：进程崩溃后由 `SessionCommit` 队列任务续跑 Phase 2；没有独立 redo log 记录中间步骤。
- **机制**：task 持久化 + 处理器读取 archived messages；队列语义"至少一次"。
- **evidence**：EV-17
- **links**：`dependency→EK-08`、`contrast→EK-05`（同为"入队异步"，但会话任务不可重算）、`causal→EK-23`（失败处理语义由消费端决定）
- **layer**：L2 · **value**：A · **epistemic**：Observation

### EK-10 归档把「可重算」与「不可重算」的输入分离
- **claim**：Phase 2 基于**已发布**的 archive 消息工作；retained 只是 active 原文，不在 Phase 2 的抽取输入里重复计。
- **机制**：archived/retained 切分；archive number 单调分配；memory_diff 只对不同项生成。
- **evidence**：EV-18
- **links**：`constraint→EK-08`、`causal→EK-21`（diff 驱动经验替代）、`contrast→EK-05`（资源可重建、会话不可）
- **layer**：L2 · **value**：B · **epistemic**：Observation

---

## C. 记忆写入（EK-11 ~ EK-18）

### EK-11 记忆类型是声明式 schema，不是硬编码分支
- **claim**：一个 memory_type = 一个 YAML（目录、文件名模板、embedding 模板、overview 模板、字段 + merge_op）。新增类型不改代码。
- **机制**：`builtin/*.yaml` 13 个模板；`memory_type_registry` 扫描并注册；`schema_model_generator` 生成模型。
- **evidence**：EV-21 / EV-22 / EV-34
- **links**：`causal→EK-12`（字段声明决定合并语义）、`mechanism→EK-19`（目录分离同源）、`constraint→EK-35`（隐私占位符必须落在字段层）
- **layer**：L2 · **value**：A · **epistemic**：Fact

### EK-12 合并语义按字段声明，而不是按整文件覆盖
- **claim**：`immutable` 身份字段不可覆盖；`replace` 整段替换；`patch` 结构化补丁；`sum` 累加；`link_merge` 双向去重。
- **机制**：`MergeOpFactory.from_field(field)` 逐字段执行；失败保留旧值并 `continue`。
- **evidence**：EV-25 / EV-26
- **links**：`dependency→EK-11`、`causal→EK-13`（字段级合并需要重读基线）、`contrast→EK-14`（删除守卫则是整条粒度）
- **layer**：L2 · **value**：A · **epistemic**：Fact

### EK-13 每次 upsert 都从磁盘重读，构建「读—改—写」而非内存态合并
- **claim**：`_apply_upsert` 每处理一条都重读最新文件，保证同批连续 patch 互相可见；这是用 I/O 换正确性。
- **机制**：读失败分类 `MemoryReadError`，只对 retryable 重试 1 次。
- **evidence**：EV-27
- **links**：`causal→EK-12`、`contrast→EK-05`（资源导入用快照，记忆更新用重读）、`constraint→EK-06`（并发下若无锁，重读仍可能竞态）
- **layer**：L2 · **value**：A · **epistemic**：Fact

### EK-14 删除发生在所有写入成功之后，且有七类跳过条件
- **claim**：删除是"最后一步"，只有目标写入成功、链接迁移成功、非同名替换、非 unresolved 时才执行。
- **机制**：① 批级 errors → 整批返回；② **批内存在 unresolved upsert → 整批跳过 delete（`has_unresolved_upserts`，独立审计补正）**；③ URI migration 失败 → 整批返回；④ 单条 upsert 异常 → 跳过该条；⑤ 同名 delete+upsert → 视为 Update；⑥ `resolution_skip` → 报告型跳过；⑦ 链接迁移失败 → 回滚 remap 并跳过 delete。
- **evidence**：EV-29 / EV-30 / EV-31 / EV-32
- **links**：`causal→EK-12`、`causal→EK-15`、`causal→EK-16`、`causal→EK-17`、`contrast→EK-02`（存储层删除是"先索引"，记忆层删除是"后删除"）
- **layer**：L2 · **value**：A · **epistemic**：Fact

### EK-15 同一批次里「删除 + 同名 upsert」被显式降级为 Update
- **claim**：不是两条独立操作，而是一次内容替换；日志 `"Replace-with-same-name treated as Update"`。
- **机制**：删除列表在构建时剔除同名项；版本号仍递增。
- **evidence**：EV-30
- **links**：`dependency→EK-14`、`mechanism→EK-12`
- **layer**：L1 · **value**：B · **epistemic**：Fact

### EK-16 无法解析的 upsert 不丢弃也不报错，而是"批级 + 条级"两层跳过
- **claim**：存在 **批级** 守卫（任一批内 upsert 未解析 → 整批 delete 全部跳过，日志 `"Skipped delete because batch contains unresolved upsert URIs"`）与 **条级** `resolution_skip`（INVALID_PEER_ID / INVALID_RANGES / PAGE_ID_TYPE_MISMATCH）两层。
- **机制**：批级守卫优先，避免"写了新对象却删了旧对象"的半完成状态；条级跳过项进入报告，不影响其它条目写入。
- **evidence**：EV-31 / EV-31b（独立审计补正，`memory_updater.py` 删除循环前）
- **links**：`dependency→EK-14`、`contrast→EK-13`（读失败重试，解析失败跳过）
- **layer**：L1 · **value**：B · **epistemic**：Fact

### EK-17 链接迁移失败会回滚 remap，并取消对应的源删除
- **claim**：`failed_link_migration_sources` 在重置前计算；失败源不删，避免"内容已迁移但链接断"。
- **机制**：先 remap links/backlinks，失败则恢复原 URI；delete 清单剔除该源。
- **evidence**：EV-32
- **links**：`causal→EK-14`、`mechanism→EK-12`
- **layer**：L2 · **value**：B · **epistemic**：Fact

### EK-18 向量化在文件写成功之后，且会跳过 sidecar 与已删 URI
- **claim**：索引永远由"当前磁盘内容"生成，而不是由内存中的 patch 结果生成；同批删除的 URI 不再向量化。
- **机制**：`_vectorize_memories` 在 delete 之后调用；重读文件 → embedding；跳过 `.overview.md`/`.abstract.md`。
- **evidence**：EV-33
- **links**：`causal→EK-01`、`contrast→EK-02`（写路径与删路径方向相反，但都保证索引不领先于源）、`dependency→EK-13`
- **layer**：L2 · **value**：A · **epistemic**：Fact

---

## D. 记忆内容设计（EK-19 ~ EK-23）

### EK-19 记忆按"语义角色"分目录，而不是按"写入时间"追加日志
- **claim**：preferences/profile/identity/soul/entities/events/experiences/cases/trajectories/skills/tools 各自目录；同一 topic 一个文件，天然支持精确读取与覆盖。
- **机制**：`directory` + `filename_template` 由 YAML 决定；用户隔离经 `user_space`。
- **evidence**：EV-21 / EV-22
- **links**：`mechanism→EK-11`、`contrast→graphiti 的时序图（corpus 外部）`、`causal→EK-20`（经验需要独立命名空间）
- **layer**：L2 · **value**：A · **epistemic**：Fact

### EK-20 `experiences` 被强制蒸馏成可执行的三段式程序
- **claim**：经验不是"发生了什么"的叙述，而是 `Situation / Approach / Reflect` 的机器指令；要求去除实体与噪声、保留 IF/THEN/ELSE、单意图原子化、Approach ≤8 bullets。
- **机制**：prompt 直接写进 YAML 的字段描述，输出会注入 agent system prompt。
- **evidence**：EV-23
- **links**：`causal→EK-21`（可替代的前提是"同一意图"）、`constraint→EK-19`（独立目录 + 独立命名）、`contrast→EK-21`（可替代 vs 不可替代的边界）
- **layer**：L2 · **value**：A · **epistemic**：Fact

### EK-21 `supersedes` 只允许"更宽的新经验"替代"更窄的旧经验"
- **claim**：新经验名不同且更宽时，才填 `supersedes`；是否删除由最终 merge 决定，写入方不直接删。
- **机制**：`supersedes` 是 `replace` 字段；删除决策后置到 merge。
- **evidence**：EV-24 / EV-32
- **links**：`dependency→EK-14`、`mechanism→EK-20`、`causal→EK-10`（diff 决定候选）
- **layer**：L2 · **value**：B · **epistemic**：Fact

### EK-22 agent-evolution 开关在"类型允许集"层裁剪，而不是在 prompt 层劝退
- **claim**：关闭时直接从允许类型中剔除 `cases/trajectories`（以及依赖它们的 experiences 走另一分支），返回 `reason=agent_evolution_disabled`。
- **机制**：`AGENT_EVOLUTION_MEMORY_TYPES` 常量 + `extract_long_term_memories` 分支。
- **evidence**：EV-20
- **links**：`causal→EK-19`、`constraint→EK-23`
- **layer**：L1 · **value**：B · **epistemic**：Fact

### EK-23 记忆抽取失败默认静默降级为空结果
- **claim**：`strict_extract_errors=False`（默认）时，抽取异常只记 warning 并返回空 memory diff；commit 仍成功。
- **机制**：外层 `except` 捕获 `"V3 memory extraction failed; returning empty result"`。
- **evidence**：EV-19
- **links**：`constraint→EK-08`（Phase 2 不阻塞）、`contrast→EK-24`（意图分析失败是硬抛错）、`causal→EK-09`（队列任务标记成功但无产出）
- **layer**：L2 · **value**：A（失败语义） · **epistemic**：Fact

---

## E. 检索（EK-24 ~ EK-29）

### EK-24 意图分析是硬依赖：没有 planner 直接 ValueError，解析失败直接抛错
- **claim**：检索前必须先由 LLM 产出 typed queries；无降级路径、无关键词 fallback。
- **机制**：`IntentAnalyzer(query_planner)`；`{"reasoning","queries"}`；非法 context_type 回退 RESOURCE。
- **evidence**：EV-12 / EV-13
- **links**：`causal→EK-25`（查询决定召回）、`contrast→EK-23`（同类失败，相反策略）、`constraint→EK-29`
- **layer**：L2 · **value**：A · **epistemic**：Fact

### EK-25 「Hierarchical」检索实际是单次向量召回 + 可选 rerank，不是递归下钻
- **claim**：`HierarchicalRetriever` 一次 embedding、一次向量检索、按 URI 取最高分去重；rerank 放大 2 倍候选；没有逐层 tree walk。
- **机制**：collection 缺失早返回空；image_query 无 level → `[2]`；reranker 仅非 image 生效；keywords 走 `search_by_keywords_in_tenant`。
- **evidence**：EV-06 / EV-08 / EV-09 / EV-10 / EV-11
- **links**：`mechanism→EK-26`（层级感来自 tier/摘要，不来自遍历）、`causal→EK-28`（scope 决定召回集合）、`contrast→EK-27`（"层级"命名与实际机制的反差）
- **layer**：L2 · **value**：A · **epistemic**：Fact

### EK-26 检索的"层级"由返回文本的 tier 决定，且有类别级深度上限
- **claim**：默认返回 `abstract`（向量 payload，零磁盘读）；`overview` 读 sidecar；`full` 读原文。只有 `events` 允许被预算抬到 `full`。
- **机制**：`DEFAULT_TIER_BY_CATEGORY`、`DEPTH_CEILING_BY_CATEGORY={"events":"full"}`、`READ_CONCURRENCY=8`、purpose preset 分桶。
- **evidence**：EV-04 / EV-07
- **links**：`dependency→EK-25`、`causal→EK-27`、`constraint→EK-28`、`mechanism→EK-01`
- **layer**：L2 · **value**：A · **epistemic**：Fact

### EK-27 当 `abstract` 被复用为 embedding 文本时，层级包含关系会反转
- **claim**：memory 类的 `abstract` 存全文，所以 `overview` 比 `abstract` **更浅**；代码用 `FULL_BODY_ABSTRACT_CATEGORIES` 显式标注，避免"补 overview"泄露更多正文。
- **机制**：`body_for_preview` 与 `AbstractOverviewFormatError`；resource/skill 的 abstract 才是短摘要。
- **evidence**：EV-05
- **links**：`causal→EK-26`、`constraint→EK-11`（schema 决定 abstract 存什么）、`contrast→EK-01`（resource 的 abstract 才是短摘要）
- **layer**：L2 · **value**：A · **epistemic**：Fact
- **注**：这是本轮最"反直觉"的实现事实，也是最值得跨项目复用的设计教训。

### EK-28 检索范围由 context_type 的默认目录 + 显式 target 决定，peer 视图叠加
- **claim**：memory 默认 `user_root`（有 actor_peer 时加 peers）；resource 默认 `viking://resources + user_root`；skill 默认 skill targets + `AGENT_SKILLS_ROOT`。
- **机制**：`default_target_directories(ctx, context_type)`；ROOT 返回 `[]`。
- **evidence**：EV-14 / EV-15
- **links**：`constraint→EK-25`、`causal→EK-31`（ACL 过滤发生在同一 scope 上）
- **layer**：L1 · **value**：B · **epistemic**：Fact

### EK-29 collection 不存在时返回空结果而不是失败
- **claim**：空库/未建 collection 是合法状态，检索直接返回 `[]`。
- **机制**：`collection_exists_bound()` 早返回。
- **evidence**：EV-11
- **links**：`contrast→EK-24`（同一链路两种失败策略：基础设施缺失 fail-soft，模型缺失 fail-hard）
- **layer**：L1 · **value**：C · **epistemic**：Fact

---

## F. 权限、隐私与安全（EK-30 ~ EK-36）

### EK-30 ACL 的有效权限是 direct ⊕ inherited，且 restricted 只切断继承
- **claim**：`effective = direct ∪ (mode=="restricted" ? ∅ : inherited)`；inherited 始终保存父当前有效权限并持续刷新；manage 者可切 inherit↔restricted，但**不能设回 none 绕过父权限**。
- **机制**：级别 read<write<manage；mask `1/3/7`；空 grants 存 `["none"]` 且必须单独；admin 隐式 manage 不落条目；根 `viking://resources` 固定 `user:* = manage`。
- **evidence**：EV-46 / EV-47 / EV-48 / EV-49
- **links**：`causal→EK-31`、`constraint→EK-28`、`contrast→EK-34`（ACL 是资源面，root key 是身份面）
- **layer**：L2 · **value**：A · **epistemic**：Observation（文档）

### EK-31 ACL 随索引更新，读侧接受最终一致
- **claim**：同一 URI 的不同 level 记录可能短暂持有不同 ACL 版本；读用一次快照，不阻塞、不强一致。
- **机制**：检索用 `acl_mode IN [inherit,restricted]` 识别受控记录；inherit 纯靠 token，不能单靠 inherit 获得通配。
- **evidence**：EV-51 / EV-50
- **links**：`dependency→EK-30`、`mechanism→EK-01`（ACL 是索引的载荷，故继承索引一致性）、`contrast→EK-08`（会话提交反而追求发布点原子）
- **layer**：L2 · **value**：A · **epistemic**：Observation（文档）

### EK-32 ACL 默认关闭，且关闭时仍写字段但不检查
- **claim**：`acl.enabled=false` 是默认；关闭时 `_ensure_access` 短路，字段照写。默认部署等于"有 ACL 数据、无 ACL 约束"。
- **机制**：`_access.py:473` `_acl_enabled`；`is_acl_uri` 限制到 resources。
- **evidence**：EV-50
- **links**：`constraint→EK-31`、`contrast→EK-36`（同一"默认关闭"模式）
- **layer**：L1 · **value**：B · **epistemic**：Fact

### EK-33 trusted 模式不是"裸信任 header"，而是"root key 背书 + header 断言身份"
- **claim**：每个请求都必须带与 `server.root_api_key` 相等的 `X-API-Key`（`hmac.compare_digest`）；缺失/不符分别抛 `Missing/Invalid API Key in trusted mode`。header 只提供 account/user/role/groups。
- **机制**：URL 显式值与 header 不一致 → InvalidArgument；admin 路径与 key-only `/health` 返回 ROOT。
- **evidence**：EV-53 / EV-54 / EV-55 / EV-56
- **links**：`causal→EK-34`、`constraint→EK-30`（ACL 校验发生在身份解析之后）、`contrast→EK-32`（前者强制密钥、后者默认关闭）
- **layer**：L2 · **value**：A · **epistemic**：Fact

### EK-34 root key 等价于跨租户 impersonation 能力
- **claim**：因为任意 account/user 都可由 header 指定，持有 root key 的调用方可代表任意租户；trusted 模式的隔离强度=root key 的保管强度。
- **机制**：无每租户密钥；无来源 IP/mTLS 校验（代码未见）。
- **evidence**：EV-53 / EV-54 / EV-57
- **links**：`dependency→EK-33`、`constraint→EK-31`
- **layer**：L2 · **value**：A · **epistemic**：Observation
- **边界**：未见 mTLS/IP allowlist 可能是部署层职责，非代码缺陷；标为待人工审查。

### EK-35 隐私是"写时占位 + 读时恢复 + 版本快照"的三段式
- **claim**：敏感值不进入 skill/session 正文，而进入 `viking://user/{id}/privacy/{category}/{target}/`；正文存占位符；读时按 active version 恢复；upsert 是全量快照，相同则不建版本。
- **机制**：`PrivacySanitizer` / `PrivacyRestorer`；`.meta.json` + `current.json` + `history/version_N.json`；activate 只切 current，不轮换外部凭证。
- **evidence**：EV-58 / EV-59 / EV-60
- **links**：`constraint→EK-11`（字段级）、`contrast→EK-36`（隐私是应用层，加密是存储层）、`causal→EK-31`（隐私目录也在 user 空间，受 ACL 影响？resources-only，故不受 ACL）
- **layer**：L2 · **value**：A · **epistemic**：Observation（文档）
- **反例**：隐私只保护"已注册值"；未注册的自由文本仍可能泄露——见 Candidates C-04。

### EK-36 文件加密默认关闭，provider 默认 local
- **claim**：`encryption.enabled=false`、`provider="local"`；开启后由 `FileEncryptor` 做静态加密。
- **机制**：`crypto/config.py:32-43` 校验配置；`providers.py` 提供 provider。
- **evidence**：EV-61
- **links**：`contrast→EK-32`（同一默认关闭模式）、`constraint→EK-35`
- **layer**：L1 · **value**：B · **epistemic**：Fact

---

## 统计

| 指标 | 值 |
|---|---:|
| EK 总数 | 36 |
| L1（实现级） | 9 |
| L2（经验级） | 27 |
| Fact | 24 |
| Observation | 12 |
| Hypothesis | 0 |
| 平均出边 | 待 `ek-graph.md` 计算 |
| 游离 EK | 0（每条 ≥1 link） |
