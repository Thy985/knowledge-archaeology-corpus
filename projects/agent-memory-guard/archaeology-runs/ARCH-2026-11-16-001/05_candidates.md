# 05 Candidates — 未验证假设、跨项目候选、人工复核

> 原则：**Hypothesis 不冒充 Fact**；跨项目假说不写成已验证 Principle。每条候选都给出“已有证据 / 缺失证据 / 验证路径”，并回链 EK。

## 一、单项目候选（C-01…C-15；独立验证新增 C-16…C-19 见第四节）

### C-01 LlamaIndex 读路径绕过使 E 层作用域不闭合
- **类型**：已复现的边界缺口（Fact）+ 影响范围假设（Hypothesis）
- **已有证据**：`GuardedChatStore.get_messages()` 直接返回底层 store（EK-42）；测试在 outsider 身份下读取 owner 内容并断言可见（EK-43）；fake backend 复现 outsider 读取 `SECRET`。
- **缺失证据**：真实 `llama-index-core` 运行时未安装；未验证真实 `ChatMemoryBuffer`/`ChatStore` 是否把该 store 对象暴露给 agent 代码。
- **验证路径**：安装 `llama-index-core`，构造 `ChatMemoryBuffer` + 两个 agent 身份，尝试 outsider `get_messages`；同时检查框架是否把 chat_store 暴露到 tool/agent 上下文。
- **影响**：E 层“per-agent 私有命名空间”在 LlamaIndex 读路径失效；写路径仍受 ACL。
- **Epistemic**：Fact（旁路存在）+ Hypothesis（agent 可达性）。回链 EK-42/43；KO-05。

### C-02 适配器公开 `.guard` 使显式 principal 可伪造
- **类型**：Fact（属性存在/可调用）+ Hypothesis（agent 可达性）
- **已有证据**：`GuardedChatStore.guard` 返回 raw `MemoryGuard`（EK-44）；`store.guard.read(key, principal="researcher")` 返回 `SECRET`；匿名直读被拒。
- **缺失证据**：真实框架是否把 store/history 对象交给 agent；CrewAI/Agno 的对象可见性未验证。
- **验证路径**：在四个真实框架中各构造一个 agent，检查其 tool/context 是否可达 memory/history 对象；若可达，记录为“E 层失效路径”。
- **影响**：`AgentHandle` 的“不可覆写 principal”在适配器对象面前失效；in-process 身份本就是断言而非认证（EK-15），但公开 raw guard 放大了误用面。
- **Epistemic**：Fact（API 暴露）+ Hypothesis（可达性）。回链 EK-44/15；KO-05/07。

### C-03 HTTP server 的 per-agent ACL 不可用
- **类型**：代码事实 + 部署推断
- **已有证据**：server 无认证、请求模型无 principal、`/write`/`/read` 不传 principal（EK-46）；`/reset` 无认证（C-14）。
- **缺失证据**：没有实际启动 server 做端到端请求；未验证部署方是否在外层加认证代理并改代码传 principal。
- **验证路径**：启动 `amg serve`，用两个 `AccessRule` 配置（需改代码传 principal 才能测）；当前代码路径应表现为 anonymous，所有 per-agent 规则按 anonymous 决策。
- **影响**：如果部署方以为 server 继承库的 per-agent ACL，会得到错误的安全假设。
- **Epistemic**：Fact（代码路径）+ Hypothesis（部署误用）。回链 EK-46；KO-01/KO-06。

### C-04 raw guard 的内省接口未 gate，构成侧信道
- **类型**：Fact（接口无 gate）+ Hypothesis（实际泄露影响）
- **已有证据**：`list_snapshots()` 返回含 `.data` 的 Snapshot；`quarantine` 返回原值；`events()` 含 principal/metadata；`classify()/origin_task()/written_by()/verify_all()` 均可直接调（EK-55）。
- **缺失证据**：未量化这些数据中有多少是敏感内容；未验证生产代码是否把 raw guard 限制在运维进程。
- **验证路径**：构造一个 admin snapshot + quarantine，检查 outsider 通过 raw guard 读取的内容；评估是否包含原始 secret。
- **影响**：若 agent 可触达 raw guard（C-02），则即使 read gate 拒绝，仍可从 snapshot/quarantine 取得原值。
- **Epistemic**：Fact（接口）+ Hypothesis（可达后影响）。回链 EK-55/16；KO-08。

### C-05 `SourceClass` 由调用方自报，自强化检测可被抑制
- **类型**：Hypothesis
- **已有证据**：`SelfReinforcementDetector` 只在 `source_class=AGENT_AUTHORED` 时累计；`source_class` 是 `write()` 参数，调用方可传 `SYSTEM`/`USER_INPUT`（EK-31）。
- **缺失证据**：没有实际构造“agent 伪装 SYSTEM 写入”并观察计数不增长；未确认框架是否总由可信层设置 source_class。
- **验证路径**：同一 agent 连续写 10 次，分别传 `AGENT_AUTHORED` 与 `SYSTEM`，比较 detector 触发次数；检查适配器是否硬编码 source_class。
- **影响**：F 层 self-reinforcement 可能被绕过；但其他 detector 仍工作。
- **Epistemic**：Hypothesis。回链 EK-31；KO-03。

### C-06 `verified_by` 可伪造，审计归属不可信
- **类型**：Fact（无校验）+ Hypothesis（审计影响）
- **已有证据**：`verified_by` 任意字符串写入 event metadata，未与 principal/admin 比对（EK-54）；测试用 `user:alice`。
- **缺失证据**：未确认下游是否有校验逻辑；未验证生产审计是否依赖该字段。
- **验证路径**：以 worker 身份 `promote(..., verified=True, verified_by="admin")`（在允许的 class gate 下），检查 event 是否记录 admin；再查是否有任何组件消费该字段做授权。
- **影响**：如果审计/合规依赖 `verified_by`，会记录错误的批准人。
- **Epistemic**：Fact（无校验）+ Hypothesis（下游依赖）。回链 EK-54；KO-04。

### C-07 脱敏把容器压成字符串，破坏调用方 schema
- **类型**：Fact（已复现）+ Hypothesis（业务影响）
- **已有证据**：`{"token": "...", "n": 1}` → `'token: [REDACTED:github_token]\nn: 1'`，read 返回 str（EK-28）。
- **缺失证据**：未验证下游是否对 dict 做 `["field"]` 访问并因此抛错；未测试 list of dict。
- **验证路径**：在一个 expect-dict 的消费路径上写入含 secret 的 dict，触发 REDACT，观察消费方异常；对 list/dict 嵌套做矩阵测试。
- **影响**：脱敏“成功”但数据结构丢失；可能被误判为数据损坏或导致 agent 逻辑失败。
- **Epistemic**：Fact（结构变化）+ Hypothesis（业务失败）。回链 EK-28；KO-02。

### C-08 design.md 的 detector / class / read 流程与实现不一致
- **类型**：Fact（文档漂移）
- **已有证据**：design.md 列 10 detector、4 类 Classification、read 先查 quarantine；实现是 7 detector、6 类、read 不查 quarantine（EK-48）。
- **缺失证据**：未确认这些文档是否被部署方/审计方直接引用；未找到更新 issue。
- **验证路径**：grep 文档引用；对照 README/CHANGELOG；在 issue/PR 中找设计意图说明。
- **影响**：部署方可能高估检测覆盖、误解 class 语义、误以为 quarantine 会影响 read。
- **Epistemic**：Fact。回链 EK-48；KO-09。

### C-09 10 个 xfail 攻击族未被 0.4.0 覆盖
- **类型**：Fact（测试自报）+ Hypothesis（实际可利用性）
- **已有证据**：10 xfail 列出跨写拼接、time-bomb、跨 key SQL、base64、chained tool outputs、Unicode homoglyph、read-then-write taint、非英文注入、fake error injection（EK-49）；中文注入实测 allow（EK-29）。
- **缺失证据**：未逐个构造真实 payload 验证“未检测”是否等于“可利用”；部分 xfail 可能是设计上不打算覆盖。
- **验证路径**：把 10 个 xfail 逐个改成主动 PoC：写入 → 读回 → 观察是否影响 agent 行为；记录“检测失败”与“实际危害”分开。
- **影响**：F 层覆盖不是 92.5% 的“全域”，而是英文/已知形状；多语言与编码绕过是明确缺口。
- **Epistemic**：Fact（未覆盖）+ Hypothesis（可利用）。回链 EK-49/29；KO-03。

### C-10 Python 3.14 / free-threaded 的 identity 行为未验证
- **类型**：Hypothesis
- **已有证据**：CI matrix 声明 3.9–3.14；`identity.py` 使用 opcode/frame/GC finalizer；CHANGELOG 说明 3.14t 线程继承不同（EK-51/17）。
- **缺失证据**：本轮只在 3.12 跑；未在 3.9/3.13/3.14t 跑 identity context 测试。
- **验证路径**：用 uv 建 3.9、3.13、3.14t 环境，跑 `tests/test_identity_context.py`；重点测 generator 暂停、线程池、`asyncio.to_thread`、GC finalizer。
- **影响**：跨版本可能把身份泄漏到错误线程/任务，或在 3.14t 下过度继承。
- **Epistemic**：Hypothesis。回链 EK-51/17；KO-07。

### C-11 `promote()` 字符串 target 在无 access 时崩溃
- **类型**：Fact（已复现）
- **已有证据**：无 access 时 `target="verified_preference"` → `AttributeError: 'str' object has no attribute 'value'`；有 access 时先 `MemoryClass(target)` 归一化（EK-53）。
- **缺失证据**：未确认是否有调用方实际传字符串；类型签名要求 `MemoryClass`，可能被视为调用方错误。
- **验证路径**：grep 文档/示例中的 `promote(` 调用；加一个回归测试固定“字符串 target 在两种配置下行为一致”。
- **影响**：升级到 0.4.0 后，原本可用的字符串调用在无 access 配置下崩溃；属于 API 兼容性缺口。
- **Epistemic**：Fact。回链 EK-53；KO-04。

### C-12 quarantine 无容量上限、无过期策略
- **类型**：Observation
- **已有证据**：`_quarantine` 是普通 dict；无 max/ttl；`quarantine` property 返回全量（EK-37/55）。
- **缺失证据**：未做长时间运行的内存增长测试；未确认运维是否有外部清理。
- **验证路径**：循环写入 10k 个会被 quarantine 的 key，测 RSS 增长；检查是否有 CLI/管理接口清理。
- **影响**：长期运行可能内存增长；quarantine 原值长期保留，扩大泄露面。
- **Epistemic**：Observation。回链 EK-37；KO-08。

### C-13 events 无容量上限、无消费者契约
- **类型**：Observation
- **已有证据**：`_events` 是 list，`_emit` 只 append，无轮转（EK-38）；`events()` 返回拷贝。
- **缺失证据**：未测长跑内存；未确认是否默认有 handler 消费。
- **验证路径**：长跑 + 高频事件，测 list 长度/RSS；检查 `add_event_handler` 的默认使用。
- **影响**：事件是审计资产，但无上限可能 OOM；无消费者则 fail-open 可见性失效（KO-03）。
- **Epistemic**：Observation。回链 EK-38；KO-03/08。

### C-14 `/reset` 无认证可清空 guard 状态
- **类型**：Fact（代码）+ Hypothesis（部署影响）
- **已有证据**：`/reset` 端点无认证，调用 `_guard` 重置；`server.py` 明确无认证（EK-46）。
- **缺失证据**：未实际启动 server 调 `/reset`；未确认是否仅测试用途。
- **验证路径**：启动 server，匿名 POST `/reset`，观察 store/events 清空；检查路由是否在生产构建中禁用。
- **影响**：任何能访问端口的人可清空记忆/审计，造成完整性/可用性破坏。
- **Epistemic**：Fact（端点）+ Hypothesis（生产暴露）。回链 EK-46；KO-01。

### C-15 v0.4.0 tag / release commit / CHANGELOG 的版本归属不一致
- **类型**：Fact（git 证据）+ Hypothesis（发布物影响）
- **已有证据**：
  - `5a2ac20` “release: 0.4.0” 日期 2026-10-07 18:26 UTC。
  - 安全修复 commit `15b75be` / `93ad25e` 日期 2026-10-14。
  - `git merge-base --is-ancestor 15b75be b8353d6` 返回 0，说明 **当前 tag `v0.4.0`（b8353d6）包含修复**；但 `CHANGELOG.md` 仍把它放在 `[Unreleased]`，`pyproject.toml` 仍是 0.4.0。
  - 38 个 commit 在 `v0.4.0...main` 之间，绝大多数是 analytics/stats。
- **缺失证据**：未确认 PyPI 0.4.0 的构建 commit/时间；未确认 tag 是否在发布后被移动。
- **验证路径**：查 PyPI release 的 upload time 与 sdist/wheel 内容；查 GitHub tag 的创建/移动事件；对照 `5a2ac20` 与 `b8353d6` 的 CHANGELOG。
- **影响**：如果用户安装的是 0.4.0 的早期构建，可能不含 fail-closed redaction 修复；如果安装的是 tag b8353d6，含修复但版本号仍是 0.4.0，无法用版本区分。
- **Epistemic**：Fact（git 关系）+ Hypothesis（发布物差异）。回链 EK-27/50；KO-09/10。

## 二、跨项目候选（XP-01…XP-05）

### XP-01 授权判断应先于副作用（跨项目）
- **候选陈述**：在 agent memory / tool runtime 中，授权检查应位于任何 store 写入、快照、检测器执行之前；拒绝路径应零副作用且不泄露存在性。
- **AMG 证据**：KO-01，S4。
- **外部候选**：Graphiti（ARCH-2026-11-15-001）的写入管线是否也在授权前做副作用？尚未对照。
- **缺失证据**：至少 2 个独立项目的同机制证据；跨项目反例。
- **验证路径**：在 Graphiti / Letta / Mem0 的写路径上做同样的“拒绝后检查 store/snapshot/detector 日志”实验。
- **状态**：Cross-project Candidate；**不得写成 Principle**。

### XP-02 作用域边界等于最弱等价路径（跨项目）
- **候选陈述**：一个 memory 安全层的有效作用域，不由主 API 决定，而由所有能产生同等读/写效果的入口中最弱的一个决定。
- **AMG 证据**：KO-05，S5（Llama 旁路 + `.guard` 暴露 + 内部 `_store` 读）。
- **外部候选**：Graphiti 的 driver/schema 直连、Mem0 的 vector store 直连、Letta 的 block manager 直连，都可能是等价路径。
- **缺失证据**：未在其他项目复现“旁路入口”。
- **验证路径**：对每个项目枚举 `store.get / driver.query / _collection.search` 等等价入口，检查是否绕过其策略层。
- **状态**：Cross-project Candidate；当前只证明 AMG 一个实例。

### XP-03 可观测性接口是权限面（跨项目）
- **候选陈述**：事件日志、隔离区、快照、分类注册表应与读写同级授权；只读审计不是低权限。
- **AMG 证据**：KO-08，S3/S5。
- **外部候选**：Graphiti 的 audit/log、Letta 的 block history、Mem0 的 history 是否可被 agent 读取。
- **缺失证据**：未在其他项目检查内省接口的 gate。
- **验证路径**：对每个项目列出 `events()/history()/snapshots()/quarantine()` 类接口，检查调用者权限。
- **状态**：Cross-project Candidate。

### XP-04 安全配置无法表达时必须 fail closed（跨项目）
- **候选陈述**：配置解析器遇到当前版本无法执行的安全字段，应报错而非静默忽略；否则“配置了”会变成“全开”。
- **AMG 证据**：KO-06，S4（YAML guard 测试）。
- **外部候选**：其他项目的 policy loader 是否有 silent ignore。
- **缺失证据**：未对照其他项目。
- **验证路径**：构造未知安全字段，观察 loader 行为（报错/忽略/警告）。
- **状态**：Cross-project Candidate。

### XP-05 指标必须绑定版本、语料、作者与复现命令（跨项目）
- **候选陈述**：安全指标不是标量，而是“版本 + 语料 + 作者 + 环境 + 复现命令”的联合声明；跨版本引用必须重跑。
- **AMG 证据**：KO-10，S4（README/compliance 明确 pin 0.2.2）。
- **外部候选**：Graphiti 的性能/召回数字、其他 memory 项目的 benchmark 是否也 pin 旧版本。
- **缺失证据**：未做跨项目指标对照。
- **验证路径**：收集各项目 README 的指标与版本，检查是否可在当前 commit 复现。
- **状态**：Cross-project Candidate。

## 三、人工复核（HR-01…HR-06）

### HR-01 `.guard` 公开属性是“逃生舱”还是“误用面”？
- **问题**：`GuardedChatStore.guard`、LangChain/CrewAI/Agno 的 `.guard` 是公开 API，还是仅内部使用？如果是前者，是否应在文档中明确“拿到它等于拿到 raw authority”？
- **证据**：EK-44；C-02。
- **需要 Owner 决策**：是否改为只读代理/移除公开属性；或明确记录为“受信任代码专用”。
- **当前处理**：不修改目标仓库；记为 Candidate。

### HR-02 LlamaIndex `get_messages` 是否应走 `guard.read`？
- **问题**：当前 `get_messages` 直读底层；如果改为 `guard.read`，需要按消息索引逐条读，性能和返回结构会变。是否接受该成本？
- **证据**：EK-42/43；C-01。
- **需要 Owner 决策**：安全优先（逐条 gate）还是兼容优先（保持批量直读 + 文档声明）。
- **当前处理**：记为 Candidate/Contradiction，不擅自“修复”。

### HR-03 server 是否只用于 localhost 演示？
- **问题**：`server.py` 明确无认证；如果官方文档/CLI 允许 `--host 0.0.0.0`，是否需要认证或强制 localhost？
- **证据**：EK-46；C-03/C-14。
- **需要 Owner 决策**：加认证、加绑定限制、或明确“仅本地”。
- **当前处理**：记为 Candidate；不修改。

### HR-04 `verified_by` 是审计字段还是授权字段？
- **问题**：如果只是审计备注，应改名/文档说明“调用方自报，不认证”；如果用于合规批准链，需要签名/身份校验。
- **证据**：EK-54；C-06。
- **需要 Owner 决策**：语义定义与下游消费契约。
- **当前处理**：记为 Candidate。

### HR-05 v0.4.0 的发布物是否含 fail-closed redaction 修复？
- **问题**：release commit `5a2ac20` 早于修复；当前 tag `v0.4.0` 指向 `b8353d6`，包含修复；CHANGELOG 仍列 Unreleased。PyPI 0.4.0 构建自哪个 commit？
- **证据**：C-15；EK-27。
- **需要 Owner/Release 决策**：重新发版（0.4.1）或在 CHANGELOG 标明 tag 内容；确保 PyPI 产物含修复。
- **当前处理**：记为 Candidate + 发布风险；不修改目标仓库。

## 四、独立验证新增候选（C-16…C-19）

### C-16 LlamaIndex `set_messages` 的 guard key 与 backing index 可能错位
- **候选陈述**：`set_messages` 用 `enumerate(messages)` 生成 `llamaindex_messages.<key>.<i>`，但 blocked/quarantine 的消息被 `continue` 跳过；backing store 只收到 `screened` 列表。若第 0 条被拦、第 1 条通过，guard 存 `.1`，backing 第 0 条是原第 1 条；后续 `add_message` 用 `len(backing)` 计算 position，可能再次写 `.1`，造成 key/verdict 与消息错配。
- **已有证据**：`integrations/llamaindex.py:75-93, 95-109`（源码逻辑）。
- **缺失证据**：尚未在 fake backend 上跑“第 0 条 BLOCK、第 1 条 ALLOW，再 add_message”的完整序列。
- **验证路径**：构造该序列，检查 guard keys、backing 索引、read 返回值三者是否一一对应。
- **Epistemic**：Hypothesis（源码可推导，未运行时复现）。

### C-17 `retire_if` 的 predicate 异常被吞掉并跳过 key
- **候选陈述**：`guard.py:1170-1173` 对 predicate 异常只 `log.exception` + `continue`；该 key 既不 retire 也不上报失败，批量治理可能出现“部分成功但无回执”。
- **已有证据**：`guard.py:1155-1195`。
- **缺失证据**：未构造 predicate 抛异常的端到端测试；未检查是否有事件/返回值记录失败集合。
- **验证路径**：注入 predicate 异常，检查 retire 返回、事件日志、store 残留。
- **Epistemic**：Fact（S3，源码）/ 影响为 Hypothesis。

### C-18 `set_messages` 全被拦截时旧消息仍在
- **候选陈述**：`if screened: self._store.set_messages(...)`；若新列表全部 BLOCK/QUARANTINE，底层旧消息保持原样。调用方以为“已设置新列表”，实际读到旧内容。
- **已有证据**：`integrations/llamaindex.py:75-87`。
- **缺失证据**：未跑“全拦”序列；未确认 `drop_blocked=False` 时是否抛出。
- **验证路径**：先写两条合法消息，再 `set_messages` 全为注入消息，读取底层与 guard。
- **Epistemic**：Hypothesis（源码可推导）。

### C-19 `server /read` 的 `found=value is not None` 把“存了 None”当成“不存在”
- **候选陈述**：`server.py:330-336` 用 `found=value is not None`；若合法写入 `None`，read 返回 `found=False`，与缺失不可区分。
- **已有证据**：`server.py:312-336`；`write` 允许任意 value。
- **缺失证据**：未启动 server 实测 `write(None)` / `read`。
- **验证路径**：本地启动 server，写 `None` 与不写 key，比较 `found/blocked/value`。
- **Epistemic**：Hypothesis（源码可推导）。

### HR-06 refresh 模式应强制输出旧 KO 差异表
- **问题**：本轮 refresh 没有逐条标注 0.3.3 旧 KO-01…KO-08 的“保留/修订/失效/新增”，导致认知连续性不足。
- **证据**：旧 corpus `projects/agent-memory-guard/03-knowledge-layer/knowledge_layer.md` vs 本轮 `03`；`06_validation/independent-validation.md` BC-01。
- **需要 Owner 决策**：是否把 `refresh-delta.md` 纳入 skill 的 refresh 必选产物（P2 Skill Improvement Proposal）。
- **当前处理**：记录为 HR-06；本轮不修改生产 Skill。

## 五、候选与 KO 的边界
- C-01/C-02/C-04 是 KO-05/KO-08 的**证据缺口**，不是新 KO；它们回答“这个边界是否真的闭合”，而 KO 回答“这类边界应如何设计”。
- C-05/C-06/C-09 是 KO-03/KO-04 的**反例/削弱项**，不升维为新 Principle。
- C-15 是 KO-10 的**版本 provenance 实例**，不是独立模式。
- 所有 C/XP/HR 均保留在 Candidates，不进入 03 Knowledge Layer 的已验证 KO。
- C-16…C-19 来自 Independent Validation 的 MISSING 项，均标 Hypothesis，未升维。
