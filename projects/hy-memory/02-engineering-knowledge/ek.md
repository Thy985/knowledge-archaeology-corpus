# 02.2 · Engineering Knowledge（EK Graph 节点，26 条）

> 每条 EK 均声明 `links`（六类边）。`layer` 指五层阶梯（L1 事实 / L2 工程知识）。本层只到 L2/L3；不在此升 L4。

---

## 路由与场景（EK-01 ~ EK-06）

### EK-01 · 三值是封闭枚举，非法输入立即失败
- **category**: ENGINEERING / layer: L1
- **statement**: `scene` 只有 `chat | working | coding | None`；任何其他字符串在入口处抛 `ValueError`，不会静默落到默认链路。
- **why**: 路由错误会把整段记忆写进错误命名空间，事后难以迁移；在入口失败比在下游写错更便宜。
- **evidence**: F-01；`client.py:880-890`
- **links**: `mechanism` → EK-02；`subsystem` → EK-02, EK-03, EK-05
- **epistemic**: Fact · **strength**: S3 · **value**: B

### EK-02 · 无 tool 的写入用 O(1) 短路，绝不付分类成本
- **category**: RUNTIME / layer: L2
- **statement**: 只有 messages 且含 tool 消息时才进入 coding 判定；纯文本 / 无 tool 对话直接走 chat，不调用 LLM。
- **why**: 分类是一次完整 LLM 调用；普通闲聊占写入量的大头，若每次分类，成本会数倍于抽取本身。
- **evidence**: F-03；`client.py:2949-2952`
- **links**: `causal` → EK-03；`constraint` → EK-03（分类只在含 tool 时发生）
- **epistemic**: Fact · **strength**: S3 · **value**: A

### EK-03 · 场景判定用单次 LLM 三分类，复用主模型而非独立 classifier
- **category**: AGENT / layer: L2
- **statement**: `classify_messages_scene` 用 `extract_tool_summary + 对话轮次` 组 prompt，复用 `LLMConfig.model`，输出 `{scene, reason}`。
- **why**: 独立 classifier 会引入第二套模型配置与部署面；场景分类与抽取共用同一模型可减少运维复杂度。代价是分类质量受主模型能力约束。
- **evidence**: F-04；`coding/judge.py:154-205`
- **links**: `causal` → EK-04；`subsystem` → EK-01
- **epistemic**: Fact · **strength**: S3 · **value**: B

### EK-04 · 分类失败一律降级到 chat（fail-soft，不是 fail-fast）
- **category**: RUNTIME / layer: L2
- **statement**: LLM 异常、JSON 解析失败、内部异常三条路径全部 `defaulting to chat`；调用方再把 `is_coding` 置 False。
- **why**: 写入不能因为分类器故障而丢数据；chat 是最保守、最不可能误用生产力 schema 的链路。代价是"用户以为写了 coding memory，实际落在 chat 层"成为可达路径。
- **evidence**: F-05, F-06；`judge.py:215-230`；`client.py:3001-3005`
- **links**: `constraint` → EK-02（分类失败时退回到 O(1) 链路的同一终点）；`contrast` → EK-13（reconcile 是 fail-closed）
- **epistemic**: Fact · **strength**: S3 · **value**: A

### EK-05 · 显式 coding 但输入无 tool：回退 chat，不制造空记忆
- **category**: RUNTIME / layer: L2
- **statement**: `scene="coding"` 是"强制走生产力链路"，但前置条件有两层：① `coding_available`（mode≠lite、messages、coding 配置存在、`coding.enable`）；② messages 含 tool。第一层不满足时**静默回退 chat，无任何日志**；第二层不满足时 warning + 回退。
- **why**: 生产力抽取依赖 tool trace（文件/命令/结果）；没有 trace 时抽取规范无输入可依，强行写只会产生空洞 fact。功能开关关闭时则根本不进入该分支——这是产品级禁用，不是输入错误。
- **evidence**: F-07；`client.py:2954-2960`（四重 AND）、`:2955-2965`（无 tool 回退）、`:2976-2978`（elif 条件）
- **links**: `contrast` → EK-02（自动判定 vs 强制指定）；`constraint` → EK-03
- **epistemic**: Fact · **strength**: S3 · **value**: B

### EK-06 · working 与 coding 共用 pipeline/schema，靠存储后缀隔离
- **category**: ARCHITECTURE / layer: L2
- **statement**: 两者是同一 productivity pipeline 的两个 variant；差异仅三点：抽取 prompt、SQLite 文件（`coding_memory.db` vs `coding_memory_cv.db`）、向量集合后缀（`*_coding_keys/*_working_facts` vs `*_cv`）。
- **why**: 复用可避免维护两套抽取/落库/链接逻辑；隔离必须落到**物理存储**，否则 `add(working)` 后 `search(coding)` 会读到 working 的库，静默读错。
- **evidence**: F-09；`client.py:3010`；`README.md:226-234`
- **links**: `subsystem` → EK-15, EK-17, EK-22
- **epistemic**: Fact · **strength**: S3 · **value**: A

---

## 写入容错与版本化（EK-07 ~ EK-11）

### EK-07 · L1 raw 先落库，持久化失败不阻断整次写入
- **category**: RUNTIME / layer: L2
- **statement**: embed 后立即 upsert L1，异常只记 `_l1_error` 并继续；后续 agent/reconcile 仍可运行（但 agent 分支要求 `vector_store is not None and memory_id`）。
- **why**: 原始对话是最不可重建的资产；派生事实可以重跑，原文丢了就永久丢失。因此原文层用"尽力写"，派生层用"要么全成要么全不成"。
- **evidence**: F-10；`writer.py:2660-2705`
- **links**: `causal` → EK-08；`contrast` → EK-08（同一写入内两种失败策略并存）
- **epistemic**: Fact · **strength**: S3 · **value**: A

### EK-08 · reconcile 失败时整批不落库，不做 ADD-all 兜底
- **category**: ENGINEERING / layer: L2
- **statement**: `reconcile` 返回 `success=False` → 返回 `ReconcileFailure`，本次一条新记忆都不写；调用方重跑整段。
- **why**: ADD-all 兜底会把"未判重的新事实"直接写进库，制造重复与矛盾；重跑的成本低于污染版本链的成本。前提是写入幂等/可重放（见 EK-28）。
- **evidence**: F-13；`writer.py:1298-1315`
- **links**: `contrast` → EK-07；`dependency` → EK-28
- **epistemic**: Fact · **strength**: S3 · **value**: A

### EK-09 · 删除是逻辑删除：SHADOW + is_latest=False
- **category**: MEMORY / layer: L2
- **statement**: DELETE 不物理删，节点保留在 VDB，状态置 SHADOW、`is_latest=False`，不参与召回；低置信度推断也复用 SHADOW。
- **why**: 记忆系统的"删除"往往是"当前不成立"而非"从未发生"；保留节点才能回溯为什么改口，也避免误删不可恢复。
- **evidence**: F-14；`reconciler.py:18-26`；`memory.py:121-128`
- **links**: `mechanism` → EK-10；`causal` → EK-12（SHADOW 不进入召回层）
- **epistemic**: Fact · **strength**: S3 · **value**: A

### EK-10 · 判重只针对链头，并补全整条 supersede 链
- **category**: MEMORY / layer: L2
- **statement**: 候选检索只取 `is_latest=True` 的节点；`_trace_full_chain` 沿 supersedes/superseded_by 补全，以链头（`chain[0]`）为代表。
- **why**: 对历史版本判重会反复制造"同一事实的多个影子"；链头代表当前语义，链只用于解释"怎么变来的"。
- **evidence**: F-15；`reconciler.py:634-658`, `:987`
- **links**: `dependency` → EK-09
- **epistemic**: Fact · **strength**: S3 · **value**: A

### EK-11 · 判重的分批维度由 owner 决定
- **category**: MEMORY / layer: L2
- **statement**: `group_items_by_owner`：`owner=user` 的记忆跨 agent 合并判重；`owner=agent` 的按 agent 隔离；只对 L2_FACT / L4_IDENTITY 判重。
- **why**: "关于用户的事实"不应因来自不同 agent 而重复；"agent 自己的知识"则不应跨 agent 合并，否则会串味。
- **evidence**: F-16；`reconciler.py:763-800`
- **links**: `mechanism` → EK-25（隔离维度设计）；`subsystem` → EK-12
- **epistemic**: Fact · **strength**: S3 · **value**: A

---

## 检索（EK-12 ~ EK-13）

### EK-12 · 召回层是白名单，L1 原文永不召回
- **category**: MEMORY / layer: L2
- **statement**: 主 VDB 召回固定 L2/L3/L4；L0 走独立隐藏通道（上限 10，避免与主通道双重召回）；L1/L5 不在召回集合；L7 由图通道负责。
- **why**: 原文是证据不是答案；直接召回原文会让模型看到未消解的冲突与噪音。分层召回保证"返回的是已加工的结论"。
- **evidence**: F-18；`reader_base.py:136-155`
- **links**: `subsystem` → EK-13；`constraint` → EK-09（SHADOW 节点被状态过滤）
- **epistemic**: Fact · **strength**: S3 · **value**: A

### EK-13 · 图重排是可选增强，失败不阻断主检索
- **category**: RUNTIME / layer: L2
- **statement**: `_apply_graph_rerank` 异常只 warning（`graph rerank failed (non-fatal)`）；图配额只在启用时生效。
- **why**: 图是增强通道，不是唯一真相源；图后端不可用不应让整个 recall 失败。
- **evidence**: F-19；`reader_base.py:653`
- **links**: `dependency` → EK-12（主通道不依赖图结果）
- **epistemic**: Fact · **strength**: S3 · **value**: B

---

## System 2 与成本控制（EK-14）

### EK-14 · System 2 手动触发、全量扫描、非幂等
- **category**: AGENT / layer: L2
- **statement**: `evolve()` 仅 `mode="ultra"`；不在 add 自动触发、无进程内队列；每次对 user 的 VDB 全量跑 Schema 归纳 + cross-domain sweeper。
- **why**: Schema 归纳需要跨大量事实看模式，增量触发会得到碎片化 schema；全量扫描成本高，必须由调用方在"批量 add 之后"显式控制。
- **evidence**: F-20；`client.py:2292-2332`
- **links**: `contrast` → EK-07（System 1 每次自动 vs System 2 手动）；`subsystem` → EK-06
- **epistemic**: Fact · **strength**: S3 · **value**: A

---

## Working / Coding 双层（EK-15 ~ EK-16）

### EK-15 · 事实保底 + 经验增值，双向 id 链
- **category**: MEMORY / layer: L2
- **statement**: 每 session 恰 1 条 `WorkingFact`（无阈值）；`Experience` 0~N（需过 value bar）；两者用 `derived_experience_ids` ↔ `source_fact_ids` 双向链接，`_crosslink` 取并集写回。
- **why**: "做过什么"必须可追溯，否则任务轨迹消失；"学到什么"允许为空，否则会为凑数生成低质经验。双向链让"这条经验从哪次任务来"与"这次任务产生了哪些经验"都可查。
- **evidence**: F-21, F-22；`store.py:73-120`；`coding/writer.py:290-325`
- **links**: `causal` → EK-16；`subsystem` → EK-06, EK-17
- **epistemic**: Fact · **strength**: S3 · **value**: A

### EK-16 · 经验阈值在抽取判断里，不是硬编码常数
- **category**: MEMORY / layer: L2
- **statement**: 没有 experience draft 时日志为 `no drafts produced (value bar)`；阈值由抽取模型的判断产生，代码中未见数值常量。
- **why**: "值得复用"是语义判断，固定余弦/长度阈值无法表达；交给抽取模型可随任务类型调整。代价是阈值不可审计、不可复现。
- **evidence**: F-23；`coding/writer.py:188`
- **links**: `constraint` → EK-15（value bar 决定 experience 数量）
- **epistemic**: Fact（行为）/ Observation（"无数值常量"基于本轮 grep 范围） · **strength**: S3 · **value**: B

### EK-29 · coding 的 `memory_id` 只暴露 Experience，不暴露 WorkingFact
- **category**: ENGINEERING / layer: L2
- **statement**: `resp["memory_id"] = experience_ids[0] if experience_ids else None`；WorkingFact 即使写入成功也不体现在 `memory_id`。因此 `memory_id is None` 不等于本次未写入。
- **why**: 对外契约沿用 chat 的"单数 memory_id"形状（chat 每轮一条节点），而 productivity 一次 add 可产生 0~N 条 experience；用首个 experience 充当 memory_id 是形状兼容，不是语义等价。
- **evidence**: `client.py:3055-3060`
- **links**: `constraint` → EK-15（返回值只覆盖双层中的上层）
- **epistemic**: Fact · **strength**: S3 · **value**: B

---

## 代码版本演化链（EK-17 ~ EK-22）
### EK-17 · 版本链零 LLM、零 embedding，纯 SQLite
- **category**: MEMORY / layer: L2
- **statement**: `HyCodeMem.observe()` 只做正则解析、git 调用与 sqlite 写入；不调用模型、不生成向量。
- **why**: 链要记录"真实 diff"，任何模型改写都会引入失真；确定性存储也让它可在模型不可用时继续积累。
- **evidence**: F-24；`chain_store.py:227-240`
- **links**: `subsystem` → EK-06；`mechanism` → EK-22
- **epistemic**: Fact · **strength**: S3 · **value**: A

### EK-18 · 隔离键必须包含 user 与 workspace，否则跨用户串链
- **category**: PERMISSION / layer: L2
- **statement**: `owner = "<user_id>|<workspace_id>"` 进入 `thread` 键与所有查询；注释明确不加 owner 会让不同用户/工作区的同名 branch/file/pkey 串成一条 thread。
- **why**: 业务键（branch/file/symbol/pkey）本身不含用户维度；记忆系统的"同名"在不同用户间是常态，不是异常。
- **evidence**: F-25；`chain_store.py:182-200`
- **links**: `constraint` → EK-25；`mechanism` → EK-25, EK-26
- **epistemic**: Fact · **strength**: S3 · **value**: A

### EK-19 · patch 三来源与"合成 diff 只按 symbol 归并"
- **category**: ENGINEERING / layer: L2
- **statement**: ① 对话里的 `diff --git` 直接截取；② `git show --unified=3`；③ 从 `old/new` 编辑动作合成。合成 diff 行号一律 1，因此按 symbol 归并，不按行区间。
- **why**: Git 反查会落空（commit 为空或被 checkout 冲掉）；合成 diff 能救回这些轮次，但行号不可信，若按行区间分链会把同一符号拆成多条假链。
- **evidence**: F-26；`chain_store.py:314-360`, `:254-262`
- **links**: `causal` → EK-17；`constraint` → EK-20
- **epistemic**: Fact · **strength**: S3 · **value**: A

### EK-20 · 失败的编辑不入链：宁可漏收，不要误收
- **category**: ENGINEERING / layer: L2
- **statement**: `_edit_failed()` 在编辑动作后窗口内查失败迹象；只有"有失败且其后无成功"才拒收；先失败后重试成功仍收。
- **why**: 链中一个未生效的版本会误导未来 agent 以为某版存在；漏收只损失覆盖率，误收会污染事实。
- **evidence**: F-27；`chain_store.py:369-384`
- **links**: `constraint` → EK-19；`contrast` → EK-21
- **epistemic**: Fact · **strength**: S3 · **value**: A

### EK-21 · git 不可用时静默返回空串，链退化为"无 patch"
- **category**: RUNTIME / layer: L2
- **statement**: `_git()` 在无 repo、非零退出、超时或异常时统一返回 `""`；调用方据此走编辑动作合成或直接跳过本轮。
- **why**: 链是附加层，不能因为 git 环境问题让 add 失败。代价是链缺失无告警，只有日志层面的间接迹象。
- **evidence**: F-28；`chain_store.py:386-394`
- **links**: `contrast` → EK-20（一个拒收、一个静默放行）；`dependency` → EK-17
- **epistemic**: Fact · **strength**: S3 · **value**: B

### EK-22 · 链是检索结果的装饰器，不是第二检索源
- **category**: MEMORY / layer: L2
- **statement**: 向量层决定返回哪些 fact；链只在 add 成功后 best-effort 挂载（`success=False` 时跳过），搜索侧传 `scene="coding"` 才展开链块。
- **why**: 两条独立检索通道会产生两套排序与冲突；把链作为"附件"可保证排序权威唯一，同时补回摘要丢掉的信息。
- **evidence**: F-29；`client.py:3030-3045`；`README.md:226-244`
- **links**: `dependency` → EK-17；`causal` → EK-07（写入成功是挂链前置）
- **epistemic**: Fact · **strength**: S3 · **value**: A

---

## 治理与隔离（EK-23 ~ EK-26）

### EK-23 · 管理面与记忆权威分离
- **category**: ARCHITECTURE / layer: L2
- **statement**: MemHub 只做 harness 授权、导入、队列、配置、健康；不存正文、不自判 supersede、不维护第二份实体；只可持久化 `memory_id` 外键。
- **why**: 两份正文必然分叉；一旦产品层能自判 supersede，SDK 的版本链就失去权威性。把产品层压成"管理面"是用架构约束换一致性。
- **evidence**: F-30；`memhub/docs/ARCHITECTURE.md`
- **links**: `constraint` → EK-15, EK-17（产品层不得复制这两层权威）
- **epistemic**: Fact（文档约束 + API 实现一致） · **strength**: S3 · **value**: A

### EK-24 · 授权绑定路径身份，而不是路径字符串
- **category**: PERMISSION / layer: L2
- **statement**: `source_grants` 存 `requested_path` 与 `root_identity`；读取时若 `root_identity_matches()` 失败，抛 `SOURCE_IDENTITY_CHANGED`；撤销是写 `revoked_at` 软删。
- **why**: 同一路径字符串可能被替换成另一个目录（挂载、重建、软链）；只校验字符串等于把旧授权永久传给新内容。
- **evidence**: F-31；`grants.py:143-170`
- **links**: `constraint` → EK-25, EK-26
- **epistemic**: Fact · **strength**: S3 · **value**: A

### EK-25 · 用户隔离靠字符集 + lane key 双保险
- **category**: PERMISSION / layer: L2
- **statement**: `user_id` 正则 `^[A-Za-z0-9._-]{1,64}$` 明确排除 `:`（SDK 用 `:` 拼 history key）；lane key 用 `@` 连接 `source@user`；仅 `hermes` 允许第二用户根。
- **why**: 隔离失败往往不是权限检查缺失，而是**键拼接歧义**：`a:b` 与 `a` + `b` 无法区分。字符集约束把歧义消灭在入口。
- **evidence**: F-33；`user_scope.py:20-50`
- **links**: `mechanism` → EK-18, EK-26；`constraint` → EK-11
- **epistemic**: Fact · **strength**: S3 · **value**: A

### EK-26 · workspace scope 只能由"所在目录"解析，不能凭空声明
- **category**: PERMISSION / layer: L2
- **statement**: scope 是 `workspace:v1:<sha256>`；`workspace_resolution` 把调用方 cwd 翻译成各 source 会记录的身份（路径哈希或 Cursor 的截断名比对）。
- **why**: 若允许任意传入 scope，调用方就能枚举/访问别人的工作区；限定"只能解析自己所在目录"把能力边界等同于文件系统权限。
- **evidence**: F-34；`workspace_resolution.py:1-45`
- **links**: `dependency` → EK-24
- **epistemic**: Fact · **strength**: S3 · **value**: A

### EK-27 · 本地 API 的 loopback 判定必须解析 IP，不能做前缀匹配
- **category**: SECURITY / layer: L2
- **statement**: Origin 白名单 + `ipaddress.ip_address(host).is_loopback`；注释点名 `127.attacker.example` 可经 DNS rebinding 解析到 127.0.0.1；Bearer 用 `hmac.compare_digest`。
- **why**: 前缀匹配把"以 127. 开头的任意域名"当成环回，等于把本地 API 暴露给任意网页。
- **evidence**: F-32；`auth.py:33-75`
- **links**: `dependency` → EK-23（管理面必须本地可信）
- **epistemic**: Fact · **strength**: S3 · **value**: B

### EK-28 · 写入幂等依赖 request_id，而非内容去重
- **category**: ENGINEERING / layer: L2
- **statement**: `request_id` 由调用方显式传入或生成 uuid；贯穿 pipeline log 与落库；`finalize_session` 对同一 user/agent/session 幂等 upsert 一个确定性节点。
- **why**: reconcile 失败后要求"调用方重跑整段"（EK-08），因此重跑必须可识别；用 request_id 而不是内容哈希，避免"内容相同但语义不同"被误判重复。
- **evidence**: `client.py:1724-1810`, `:2966-2968`
- **links**: `dependency` ← EK-08；`mechanism` → EK-15（episode upsert 同源思路）
- **epistemic**: Fact（机制存在）/ Observation（"重跑依赖 request_id 保证幂等"为组合推断） · **strength**: S3 · **value**: B
