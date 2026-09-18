# Engineering Knowledge — EK Graph（hermes-agent @ fba4cb1）

> 每条 EK 声明 links（六类边：mechanism/subsystem/causal/dependency/constraint/contrast）。
> 证据编号引用 F-xx（见 01_project-layer/project-facts.md）。

## 一、记忆系统（Memory）

**EK-01 冻结快照记忆契约**
记忆（MEMORY.md/USER.md）在 session 起始冻结为系统提示词快照；会话中写盘不改变 prompt。
- Evidence: F-03/F-04/F-22
- links: mechanism→EK-02（都是 prompt 一致性机制）; subsystem→EK-05; constraint→EK-22（prompt cache 神圣性约束快照设计）
- Value: A

**EK-02 prompt cache 是记忆设计的因**
"Per-conversation prompt caching is sacred" 是 AGENTS.md 明文不变量；记忆冻结、skills 变更 deferred、工具集不变——全部服务于 prefix cache 复用。
- Evidence: F-22/F-03/F-06
- links: causal→EK-01（cache 神圣性→冻结快照设计）; causal→EK-03（cache 神圣性→变更 deferred）; constraint→EK-23（narrow waist）
- Value: A

**EK-03 cache-aware 变更纪律**
改变系统提示状态的斜杠命令默认 deferred（下个 session 生效）+ opt-in `--now`；唯一例外 context compression。
- Evidence: F-22
- links: mechanism→EK-02; contrast→EK-28（letta 的 post-turn push：写后即生效 vs hermes 的 next-session）
- Value: A

**EK-04 记忆文件 = 双通道声明**
MEMORY.md（agent 笔记）+ USER.md（用户画像）双文件；learning_graph 按 § 拆卡片、`memory:<source>:<index>` 引用。
- Evidence: F-05
- links: subsystem→EK-05; mechanism→EK-06（都是记忆的结构化存储）
- Value: B

**EK-05 记忆三层分工**
系统提示词冻结快照（权威视图）+ memory 工具（mid-session 写盘）+ MemoryManager/provider（外部 provider 扇出）——三通道各司其职。
- Evidence: F-03/F-04/F-25/F-26
- links: causal→EK-25（冻结快照→provider 只做侧通道）; subsystem→EK-04; dependency→EK-26（provider 生命周期）
- Value: A

**EK-06 记忆外部化 = 单外部 provider 约束**
MemoryManager 只允许一个外部插件 provider 注册（tool-schema bloat、conflicting backends）；内置 provider 恒允许。
- Evidence: F-26/F-25
- links: constraint→EK-25; contrast→EK-24（letta 多后端可配置）；mechanism→EK-27
- Value: B

**EK-07 checkpoint API v1/v2 演进**
记忆压缩前钩子从 v1 best-effort 演进到 v2 opt-in fail-closed（normalized evidence handoff + strict-mode failure propagation）；签名内省防 TypeError。
- Evidence: F-25
- links: causal→EK-08; contrast→EK-28（失败语义：v2 严格 vs letta push 状态机）
- Value: A

**EK-08 contextvars 并发隔离**
provider 后台任务必须经 ctx_bound/spawn_context_thread 绑定调用方 contextvars；空 context 落默认 profile 或 secrets fail-closed。
- Evidence: F-27
- links: dependency→EK-07; constraint→EK-26
- Value: B

## 二、技能系统（Skills）

**EK-09 Skills = 程序性记忆**
SKILL.md 窄"how to do X"，MEMORY.md 宽声明；技能是 agent 的可执行记忆。
- Evidence: F-06
- links: contrast→EK-04（程序性 vs 声明性记忆）；mechanism→EK-10; subsystem→EK-11
- Value: A

**EK-10 技能布局与生命周期**
`<skills>/[category/]<skill>/SKILL.md` + references/templates/scripts/assets；新技能落 ~/.hermes/skills/；现有技能原位修改。
- Evidence: F-06
- links: subsystem→EK-11; dependency→EK-12
- Value: B

**EK-11 技能治理四件套**
skill_manager_tool（创建/编辑）+ skill_linter（咨询性 lint）+ skills_guard（供应链扫描）+ skill_manager_guards（pinned/curator/org-mirror/background-review 守卫）。
- Evidence: F-09/F-07/F-08
- links: mechanism→EK-10; constraint→EK-12; causal→EK-13
- Value: A

**EK-12 硬验证 vs 软 lint 分层**
frontmatter 硬验证（block 非协商项）与 lint（advisory）分层；lint 从不自行 block。
- Evidence: F-09
- links: constraint→EK-11; contrast→EK-13（扫描是信任层不是 lint）
- Value: B

**EK-13 供应链信任分级**
builtin（never scanned）/trusted（openai/anthropics/hf/NVIDIA：caution 允许）/community（findings block unless --force）/agent-created（dangerous→"ask" 报错重试）。**--force 仅覆盖 caution；dangerous+community/trusted 为 hard_block 不可覆盖（R-3）**。
- Evidence: F-07/F-08；tools/skills_guard.py:660-673
- links: dependency→EK-14; contrast→EK-29（letta pre-commit 是格式门，hermes 是信任门）
- Value: A

**EK-14 供应链扫描已知缺口（诚实声明）**
静态正则无法绑定语言写 API 与动态目标；agent-config 写入只报低级 finding；未来第四 "mechanical" tier。
- Evidence: F-10
- links: constraint→EK-13; contrast→EK-30（runtimes 沙箱可补静态缺口）
- Value: A

**EK-15 Curator 空闲维护闭环**
空闲触发（无 cron）后台维护：last run>7d 且空闲>2h → 生命周期迁移，可选 fork AIAgent pin/archive/consolidate/patch；.curator_state 持久化。
- Evidence: F-11
- links: causal→EK-16; mechanism→EK-09; subsystem→EK-11
- Value: A

**EK-16 Curator 不变量**
只动 curator-managed skills；永不删除只归档；pinned 绕过；fork 用 auxiliary client 不碰主会话 cache。
- Evidence: F-12
- links: constraint→EK-15; dependency→EK-22
- Value: A

## 三、状态存储（State）

**EK-17 SQLite WAL 状态库**
state.db：session metadata/message history/model config/FTS5；WAL 模式；parent_session_id 链压缩；source-tagged。
- Evidence: F-13
- links: subsystem→EK-18; mechanism→EK-19; causal→EK-20
- Value: A

**EK-18 WAL 兼容回退**
NFS/SMB/FUSE/WSL1/ZFS 不兼容标记 → 回退 DELETE journal（读阻塞写）；64MiB WAL limit。
- Evidence: F-14
- links: dependency→EK-17; contrast→EK-31（letta 用 git 无 SQLite WAL 面）
- Value: A

**EK-19 生产/测试实例 guard**
hermes_state_guard：BYPass env、production 判定、test instance 注册——测试实例不得触碰生产库。
- Evidence: F-15
- links: constraint→EK-17; mechanism→EK-32
- Value: A

**EK-20 回滚权威模型**
rewind：durable transcript 是权威，warm history 只需一致；carrier-aware（保留 hidden handoff scaffold）；CLI/gateway/TUI 单实现。
- Evidence: F-16
- links: causal→EK-21; dependency→EK-17; contrast→EK-33（letta 无此用户级回滚）
- Value: A

**EK-21 检查点快照（shadow git）**
文件变更前透明快照到共享 shadow git store；GIT_DIR/WORK_TREE/INDEX_FILE 隔离不污染用户项目；跨项目 blob 去重。
- Evidence: F-17
- links: causal→EK-20（可回滚的物理底座）; mechanism→EK-31（都是 git 作为基础设施）
- Value: A

## 四、安全与信任（Security）

**EK-22 唯一承重边界 = OS 隔离**
SECURITY.md 明文：唯一对对抗性 LLM 的安全边界是 OS 级隔离（terminal 容器/云沙箱/远程）；in-process 启发式非边界。
- Evidence: F-21/F-20
- links: constraint→EK-23; contrast→EK-34（letta fail-closed 内核沙箱：两项目对"承重墙"定义不同）
- Value: A

**EK-23 in-process 防御 = 协作性拒绝 + 审计**
file_safety：defense-in-depth 非边界；价值=对尊重工具错误的模型的明确拒绝 + 可见审计轨迹。
- Evidence: F-20
- links: dependency→EK-22（不替代 OS 边界）; contrast→EK-35; mechanism→EK-36
- Value: A

**EK-24 注入扫描分层**
context 文件（AGENTS.md/.cursorrules/SOUL.md）注入扫描命中即 BLOCK；scope="context"（strict 模式对克隆仓库太激进）。
- Evidence: F-18
- links: mechanism→EK-25（都是 prompt 侧防御）; contrast→EK-36; constraint→EK-22
- Value: A

**EK-25 user_authored 例外**
HERMES_HOME 用户自己 SOUL.md 命中只 WARN 仍加载（#112570）；distribution 拥有的 SOUL.md block。
- Evidence: F-19
- links: contrast→EK-24（用户文件 vs 项目文件不同策略）; constraint→EK-22; causal→EK-34
- Value: A

**EK-26 保护指令审批**
SOUL.md/config.yaml 同信任类；file-tool 写经 protected-instruction approval gate。
- Evidence: F-34
- links: dependency→EK-23; mechanism→EK-37（approval 族）
- Value: B

**EK-27 凭证池**
credential_pool.py 存在（agent/）；凭证管理属安全面（细读见后续 run）。
- Evidence: agent/credential_pool.py（存在性）
- links: subsystem→EK-23; constraint→EK-22
- Value: C

## 五、架构与工具（Architecture）

**EK-28 narrow waist 工具门槛**
每个 model tool 随 API call 发送 → 核心工具门槛高；新能力走 CLI+skill/service-gated/plugin。
- Evidence: F-23
- links: constraint→EK-02; causal→EK-38; contrast→EK-39
- Value: A

**EK-29 工具注册自动发现**
tools/*.py 顶层 registry.register()；model_tools 触发发现；handler 全返回 JSON string；check_fn TTL-cached。
- Evidence: F-24
- links: dependency→EK-28; mechanism→EK-38
- Value: B

**EK-30 多前端单核心**
CLI/gateway(~20 平台)/TUI/desktop 共享 agent 核心；扩展靠 plugins+skills 不靠核心增长。
- Evidence: F-02/F-23
- links: subsystem→EK-29; constraint→EK-28
- Value: B

**EK-31 入口 bootstrap 纪律**
cli.py 首 import hermes_bootstrap（Windows UTF-8）；HERMES_QUIET 抑制噪音。
- Evidence: F-28
- links: mechanism→EK-32; constraint→EK-30
- Value: C

**EK-32 状态/启动 watchdog**
hermes_startup_watchdog.py、hermes_state_registry/repair/portability——启动韧性族。
- Evidence: 顶层文件（存在性）
- links: subsystem→EK-17; mechanism→EK-19
- Value: C

**EK-33 轨迹压缩策略**
保护 head+尾 N 轮，中间摘要必要轮数（不拆 tool_call/response 对）；RL/benchmark 轨迹专用。
- Evidence: F-31
- links: mechanism→EK-02（都是 token 预算管理）; contrast→EK-20（压缩 vs 回滚的互补）
- Value: B

**EK-34 人格文件信任类**
SOUL.md 属用户身份文件（同 config.yaml 信任类）；项目 checkout 永不供应。
- Evidence: F-32/F-34
- links: causal→EK-25; constraint→EK-22
- Value: B

## 六、跨项目对照 EK（对比 letta-code，ARCH-2026-09-18-001）

**EK-35 记忆后端哲学对照**
letta：git-backed MemFS（记忆=文件树+git 资产）；hermes：SQLite state.db + 冻结快照 + provider 外部化。同一"记忆持久化"问题的两种实现。
- Evidence: 本 run F-03/F-17 vs letta run EK-01
- links: contrast→EK-01; contrast→EK-17
- Value: A

**EK-36 安全边界哲学对照**
letta：fail-closed 内核沙箱（seatbelt/bwrap 物理墙，无沙箱即抛错）；hermes：唯一承重边界=OS 隔离，in-process 防御明确非边界。
- Evidence: 本 run F-21/F-20 vs letta run KO-02/EK-25
- links: contrast→EK-22; contrast→EK-23
- Value: A

**EK-37 记忆写入时机对照**
letta：post-turn push（每轮后自动提交）；hermes：mid-session 写盘 + 下 session 冻结生效（cache 优先）。
- Evidence: 本 run F-03/F-22 vs letta run EK-05
- links: contrast→EK-01; contrast→EK-03
- Value: A

**EK-38 供应链安全机制对照**
letta：pre-commit frontmatter 门禁（格式/结构）；hermes：skills_guard 信任分级扫描（安全策略）——同一"外源内容进门"问题的两维。
- Evidence: 本 run F-07/F-08 vs letta run EK-15
- links: contrast→EK-13; contrast→EK-29
- Value: A

**EK-39 状态回滚能力对照**
letta：git reflog/checkout 资产级回滚；hermes：rewind（用户轮语义）+ checkpoint shadow git（文件级）。用户可逆性维度 hermes 更细粒度。
- Evidence: 本 run F-16/F-17 vs letta run EK-20
- links: contrast→EK-20; contrast→EK-21
- Value: A

**EK-40 记忆旁路面（Reconciliation R-1）**
`skip_memory`（CLI `--ignore-rules` 映射）跳过外部 provider；enabled memory toolset 仍加载 MEMORY.md；cron agents 现 run with skip_memory=False。冻结快照有显式条件旁路。
- Evidence: agent/agent_init.py:1243-1284；hermes_cli/cli_agent_setup_mixin.py:673
- links: mechanism→EK-05（同属记忆加载面）; constraint→EK-01（旁路打破冻结契约的绝对性）; contrast→EK-25
- Value: A

**EK-41 执行指导工具集中性（Reconciliation R-5）**
#39797：hard "use web_search" 曾覆盖 SOUL.md 并在 Blank Slate 悬空 → 执行指导文本保持 toolset-neutral（不指名具体工具）。
- Evidence: agent/prompt_builder.py:500
- links: constraint→EK-28（narrow waist 的工具面约束）; causal→EK-24（注入/指导治理）; contrast→EK-34
- Value: B

## EK 图统计（本 run）
- EK 总数：41（Reconciliation 后）
- 边类型分布：mechanism 13 / subsystem 14 / causal 13 / dependency 10 / constraint 18 / contrast 15（含双向，去重后 ≥65 条唯一边）
- 平均出边：≥1.5（全部 EK 均有 ≥1 links）
- 游离 EK：0
