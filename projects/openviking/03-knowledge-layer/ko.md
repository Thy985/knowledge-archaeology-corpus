# 03 · Knowledge Layer（8 个 Generalized KO）

> 每个 KO 必须：① 有 `aggregation_rule`（R1–R4）；② 列出簇内 EK 与边类型；③ 证明"解释范围扩大"；④ L3+ 至少 3 个定向反例；⑤ 标注 Epistemic Status。
> **单项目证据**：所有 L4 均标 `Cross-project validation pending`，不写成 Law。

---

## KO-01 · 索引可以落后，但不能领先于源

- **层级**：**L4 认知模型**（`knowledge_layer: generalized`）
- **scope**：仅 **indexed 命名空间**（`viking://system/...` 为 `FILE_ONLY`，不建索引、不参与本不变量；见 `core/namespace_policy.py:35-47`）
- **一句话**：在一个"源 + 派生索引"的系统中，**派生层允许缺失（漏检），但不允许指向已不存在或已变更的源（坏结果）**；所有写入/删除顺序都应朝这个方向收敛。
- **纵向链**：
  - L1：`rm` 先 `delete_uris` 再删文件；`_vectorize_memories` 在文件写成功后调用；`add_resource` 内容先落地、索引入队（EV-36 / EV-33 / EV-40）。
  - L2：因为索引是可重建的派生品，而内容是唯一权威；崩溃窗口的代价必须落在"少一条命中"，而不是"命中一个不存在的对象"（EK-01 / EK-02 / EK-18）。
  - L3：这是一种**不对称的一致性策略**：写路径"源先、索引后"，删路径"索引先、源后"——方向相反，但不变量相同（EK-02 的 contrast 边）。
  - L4：**派生系统的失败方向应由"可恢复性"决定，而不是由操作顺序的对称性决定。**
- **aggregation_rule**：**R3 不变量簇** — 簇 {EK-01, EK-02, EK-03, EK-04, EK-18}，边类型 `causal ×4 / mechanism ×3 / constraint ×1 / contrast ×1`；五条 EK 从不同操作（导入/删除/移动/向量化）汇聚到同一不变量。
- **解释范围扩大**：从"OpenViking 的 rm 顺序"扩展到"任何 read-model / 缓存 / 搜索索引 / 物化视图的更新协议"。
- **反例攻击（3）**：
  1. `mv` 复制成功、索引更新失败时清目标 → 目标不存在、源仍在，索引不领先，**符合**。
  2. `add_resource` 在语义队列未完成时即可被检索到（旧摘要）→ 索引落后于新内容，**符合**（漏检/旧检，非坏结果）。
  3. `MemoryUpdater` 先删旧 URI 索引、再写新 URI 文件；若写失败 → 旧文件在、旧索引无，**符合**。
  - **未找到反例**；但边界存在：若未来引入"索引先行、文件后写"的批量导入，则不变量会被破坏（见 C-05）。
- **Epistemic**：`Pattern (Strongly evidenced · Cross-project validation pending)`。corpus 中 mem0（向量为主）与 graphiti（图为主）未显式声明该不变量，待跨项目验证。
- **value**：A

---

## KO-02 · 删除是写入协议的最后一步，而不是独立操作

- **层级**：**L3 工程模式**（独立审计降级：当前仅 memory 写入路径有完整代码证据）
- **一句话**：在多步写入中，**删除必须由"所有前置写入成功"来授权**；任何一步失败都应让删除消失，而不是让删除先发生。
- **纵向链**：
  - L1：`apply_operations` 先 upsert、后 delete；六类跳过条件（整批 errors / URI migration / 单条异常 / 同名替换 / resolution_skip / 链接迁移失败）（EV-29~32）。
  - L2：因为删除不可逆，而写入可重试；把不可逆动作放在链尾，失败时系统只会"多留一条旧数据"，不会"少一条新数据"（EK-14）。
  - L3：这是 **compensable-first ordering**：先做可补偿动作（写），后做不可补偿动作（删）；同名替换被降级为 Update 也是同一原则的体现（EK-15）。
  - L4：**不可逆操作应当由成功的前置状态授权，而不是由调用方的意图授权。**
- **aggregation_rule**：**R2 因果链簇** — EK-14（总协议）→ EK-15（同名分支）→ EK-16（解析失败分支）→ EK-17（链接迁移分支）→ EK-21（经验替代的延迟删除）。
- **解释范围扩大**：适用于任何"更新 = 写新 + 删旧"的系统（对象存储生命周期、LSM compaction、Git rebase、数据库 MVCC 清理）。
- **反例攻击（3）**：
  1. 若 upsert 成功、delete 失败 → 旧条目残留，系统出现**重复**；这是"保守但可见"的代价，不违反原则。
  2. 若同批 delete 先于 upsert（代码顺序若被重构）→ 写失败将导致数据丢失；当前代码未如此（EV-32）。
  3. `add_resource` 覆盖同名文件时是否先清旧内容？**未验证**（见 C-01）；若先清后写，则该路径违反原则，需标为潜在反例。
- **Epistemic**：`Pattern (Single-path evidence · boundary: add_resource overwrite unverified)`。独立审计判定：不具备 L4 所需的多路径证据；L4 版本保留为 Candidate C-05。
- **value**：A

---

## KO-03 · 渐进披露的本质是"返回成本决定抽象层级"

- **层级**：**L3 工程模式**
- **一句话**：当检索结果的**返回成本**（token / 磁盘读 / 延迟）成为主约束时，系统应按"零读摘要 → 结构概览 → 原文"分层，并给每类内容设定**深度上限**，而不是让调用方自由取全文。
- **纵向链**：
  - L1：`abstract` 来自向量 payload（零读）；`overview` 读 sidecar；`full` 读原文；默认 tier=abstract，仅 events 可到 full（EV-04/05/07）。
  - L2：因为 agent 的上下文窗口是稀缺资源，默认返回全文会把"检索"退化为"搬运"；分层让 90% 的调用只付 10% 的成本（EK-26）。
  - L3：**tier + depth ceiling + purpose preset** 三件套是可复用模板：默认浅、按类别限深、按用途分桶（EK-25/26/27）。
  - L4（**未采用**）：不宜升到"所有检索系统都应分层"——这是设计选择，不是普遍定律。
- **aggregation_rule**：**R4 主题簇 + R1 机制簇** — 簇 {EK-25, EK-26, EK-27, EK-28, EK-01}；`dependency ×2 / causal ×2 / constraint ×2 / contrast ×2`。
- **解释范围扩大**：从"OpenViking 的 tier"扩展到"任何 RAG / 代码检索 / 文档问答的上下文预算控制"。
- **反例攻击（3）**：
  1. resource/skill 的 `abstract` 是短摘要，层级正常；只有 memory 类反转（EK-27）→ 说明模式**需要按内容类型校准**，不能一刀切。
  2. 调用方显式 `detail=full` 可绕过默认；深度上限只约束"自动加深"，不约束显式请求 → 模式是"默认策略"而非"硬限制"。
  3. `purpose=coding` 的 bucket preset 可能给出比 abstract 更深的组合 → 说明"成本"还受用途影响。
- **Epistemic**：`Pattern (Strongly evidenced · single-project)`。
- **value**：A

---

## KO-04 · 记忆应是"可声明 schema 的一等对象"，而不是自由文本堆

- **层级**：**L3 工程模式**
- **一句话**：把记忆类型声明为 schema（目录、文件名、embedding 文本、字段、合并语义），可以让"新增一类记忆"变成配置行为，而不是代码分支。
- **纵向链**：
  - L1：13 个 YAML 模板；`preferences` 以 user+topic 为身份键；`experiences` 有 operation_mode/agent_only/supersedes；`merge_op` 五种（EV-21~25）。
  - L2：因为记忆的"结构"决定它能否被精确更新、精确读取、精确检索；自由文本只能整体替换，schema 才能字段级 patch（EK-11/12）。
  - L3：**schema + merge_op + 文件名模板** 是可复用模板；`immutable` 字段承担"身份键"，`replace/patch` 承担"内容演化"（EK-13/19/20）。
  - L4（**未采用**）：不宜断言"所有记忆都必须 schema 化"——对话式自由记忆仍有价值。
- **aggregation_rule**：**R1 机制簇** — 簇 {EK-11, EK-12, EK-13, EK-19, EK-20, EK-21}；`mechanism ×3 / causal ×4 / constraint ×2 / dependency ×2`。
- **解释范围扩大**：适用于 agent memory、CRM 客户档案、个人知识库、配置即产品（configuration-as-product）。
- **反例攻击（3）**：
  1. `extra_fields` 接受 schema 之外的键 → schema 不是封闭的，"声明式"不等于"强校验"。
  2. legacy 记忆文件无 schema 仍可读 → schema 是写入期约束，不是读取期契约。
  3. `agent_evolution_enabled=false` 会裁剪类型 → schema 之上还有策略层（EK-22）。
- **Epistemic**：`Pattern (Strongly evidenced · single-project)`。
- **value**：A

---

## KO-05 · 两阶段提交的价值在"发布点"，不在"原子性"

- **层级**：**L3 工程模式**
- **一句话**：当第二阶段不可重算（LLM 生成不确定）时，正确做法是**把"已发布"作为唯一事实点**，让第二阶段可续跑但不可重放；不要假装它是可回滚事务。
- **纵向链**：
  - L1：Phase 1 锁内发布 archive + 入队 + retained + marker；Phase 2 锁外抽取；崩溃靠持久队列续跑；无 redo log（EV-16/17/18）。
  - L2：因为模型输出不可复现，重放会得到不同记忆；系统只能"从已发布 archive 继续"，不能"回到提交前重算"（EK-08/09/10）。
  - L3：**publish-then-process** 模式：先固化输入，再异步派生；派生失败不改变输入（EK-23 的静默降级是这一模式的代价）。
  - L4（**未采用**）：不宜升为"所有异步系统都应 publish-then-process"——可重算的派生（如索引）应选择"重算"而非"续跑"。
- **aggregation_rule**：**R2 因果链簇** — EK-08（两阶段）→ EK-09（恢复）→ EK-10（输入分离）→ EK-23（失败语义）。
- **解释范围扩大**：适用于 LLM pipeline、审计日志、事件溯源、任何"输入不可重放"的异步处理。
- **反例攻击（3）**：
  1. 若 Phase 1 的 marker 写失败，重启后可能重复归档 → 需要幂等键；代码是否幂等**未验证**（C-02）。
  2. 队列"至少一次"可能重复消费 → 依赖 memory_diff 去重与 version 递增；重复是否完全幂等**未验证**。
  3. 抽取失败静默返回空，任务却标记成功 → "发布点"正确，但"完成度"不可观测（EK-23 的副作用）。
- **Epistemic**：`Pattern (Strongly evidenced · partial: idempotency unverified)`。
- **value**：A

---

## KO-06 · 身份委派的强度，不超过背书密钥的保管强度

- **层级**：**L4 认知模型**
- **一句话**：当系统允许"上游声明我是谁"时，真正的安全边界不是 header 校验，而是**谁持有那把可以声明任意身份的钥匙**。
- **纵向链**：
  - L1：trusted 模式每个请求都要求 `X-API-Key == root_api_key`（`hmac.compare_digest`）；header 提供 account/user/role/groups；URL 与 header 不一致即拒绝（EV-53~56）。
  - L2：因为 root key 可代表任意 account/user，它等价于"跨租户 impersonation"；header 校验只防误用，不防持有者恶意（EK-33/34）。
  - L3：**钥匙分级**：api_key 模式 per-user key（窄权限）；trusted 模式 root key（宽权限）；ACL 只在资源面做二次收敛（EK-30/31/32）。
  - L4：**委派强度 = 背书密钥的权限半径 × 保管强度**；增加 header 校验不会降低这个乘积。
- **aggregation_rule**：**R3 不变量簇** — 簇 {EK-30, EK-31, EK-32, EK-33, EK-34}；`constraint ×4 / causal ×2 / contrast ×2`。
- **解释范围扩大**：适用于 SSO、JWT 网关、service mesh mTLS、CI OIDC、任何"上游注入身份"的架构。
- **反例攻击（3）**：
  1. api_key 模式不使用 root key 做数据面身份 → 模式不同，不变量仍成立（密钥权限半径更小）。
  2. 若 trusted 服务部署在 mTLS + 网络隔离后，root key 泄露概率下降 → 乘积变小，但**结构不变**。
  3. ACL 可独立于身份拒绝某资源 → 说明"身份"不是唯一授权层；但 ACL 不能阻止 root key 持有者改 ACL（admin 路径返回 ROOT）。
- **Epistemic**：`Principle (Strongly evidenced · Cross-project validation pending)`。
- **value**：A

---

## KO-07 · 写入路径乐观执行，读取路径保守降级（Disputed）

- **层级**：**L3 工程模式**
- **一句话**：同一系统可以对**写**采取"尽力而为、跳过坏条目"，对**读**采取"缺基础设施就返回空"；但**不能**对"意图理解"这种决定召回正确性的步骤也乐观——那会把错误藏进结果里。
- **纵向链**：
  - L1：单条 upsert 失败继续（EK-14）；resolution_skip 报告型跳过（EK-16）；抽取异常返回空（EK-23）；collection 缺失返回空（EK-29）；但 IntentAnalyzer 缺 planner 直接 ValueError（EK-24）。
  - L2：因为写失败可重试、可观测（version/日志），而读路径的"静默错"会直接污染 agent 决策（EK-23 vs EK-24 的对比）。
  - L3：**失败策略应按"可观测性 × 可重试性"分配**：可观测+可重试 → 乐观；不可观测 → 硬失败。
  - L4（**未采用**）：跨系统证据不足（仅 OpenViking 单例），不升。
- **aggregation_rule**：**R1 机制簇** — 簇 {EK-13, EK-14, EK-16, EK-18, EK-23, EK-24, EK-29}；`contrast ×3 / causal ×4 / dependency ×2`。
- **解释范围扩大**：适用于任何 ingestion pipeline + query serving 的双路径系统。
- **反例攻击（3）**：
  1. 抽取失败静默返回空，**不可观测**（只 warning）→ 违反"不可观测应硬失败"的自洽性，是本模式的真实反例；只能解释为"commit 可用性优先"的权衡。
  2. 意图分析失败硬抛，但调用方可能重试同一个坏 query → 用户体验差；硬失败不等于更好。
  3. collection 缺失返回空，可能让调用方误以为"没有相关知识"，同样是静默错。
- **Epistemic**：`Pattern (Disputed · 2 real counterexamples · 不用于跨项目推广)`。独立审计判定 OVER_GENERALIZED：该模式无法同时解释 intent 硬失败与两处静默失败；建议移入 Candidates 作为局部观察。
- **value**：B（有意下调：反例 1/3 直接削弱普适性）

---

## KO-08 · "文件系统即接口"是接口层隐喻，不是存储层等价

- **层级**：**L3 工程模式**
- **一句话**：用 `ls/read/write/rm/mv` 作为上下文的主接口，能换来可解释性与可编辑性，但必须**自己补齐** POSIX 不提供的语义：检索、权限、事务、版本、摘要。
- **纵向链**：
  - L1：`viking://` 三类上下文；FSService 暴露 ls/tree/stat/read/grep/glob；`find/search` 是扩展而非 POSIX；ACL/锁/事务都是自建（EV-01/14/36/46）。
  - L2：因为 agent 与人都熟悉文件隐喻，接口成本低；但文件系统本身不提供语义检索与租户隔离（EK-01/28/30）。
  - L3：**隐喻层与实现层分离**：对外是文件，对内是"内容存 + 向量索引 + 锁 + ACL + 队列"五件套（EK-05/06/26）。
  - L4（**未采用**）：不宜断言"上下文数据库都应文件系统化"——这是产品选择。
- **aggregation_rule**：**R4 主题簇** — 簇 {EK-01, EK-05, EK-19, EK-26, EK-28}；`mechanism ×2 / constraint ×2 / causal ×2`。
- **解释范围扩大**：适用于"用熟悉隐喻降低使用成本，但内部重建语义"的产品设计（如 S3 的 object 隐喻、Git 的 snapshot 隐喻）。
- **反例攻击（3）**：
  1. `find` 不是 POSIX 原语 → 说明"文件系统"只是对外接口，不是内核。
  2. 权限语义（ACL inherit/restricted）在 POSIX 中不存在 → 自建。
  3. 事务/锁协议（`.ovlock`、token、版本迁移）完全自建 → 隐喻不提供正确性。
- **Epistemic**：`Pattern (Strongly evidenced · single-project)`。
- **value**：B

---

## 汇总

| KO | 层级 | 规则 | 簇规模 | Epistemic | value |
|---|---|---|---:|---|---|
| KO-01 | L4 | R3 | 5 | Pattern (cross-project pending) | A |
| KO-02 | L3 | R2 | 5 | Pattern (single-path) | A |
| KO-03 | L3 | R4+R1 | 5 | Pattern (single-project) | A |
| KO-04 | L3 | R1 | 6 | Pattern (single-project) | A |
| KO-05 | L3 | R2 | 4 | Pattern (partial) | A |
| KO-06 | L4 | R3 | 5 | Principle (cross-project pending) | A |
| KO-07 | L3 | R1 | 7 | Pattern (disputed) | B |
| KO-08 | L3 | R4 | 5 | Pattern (single-project) | B |

- L3+ KO 全部有 ≥3 反例（共 24 条定向攻击）。
- 无 KO 使用"同子系统"作为聚合理由。
- 无 KO 达 L5（方法论）——单项目不足以上升为"如何行动"的普遍准则。
