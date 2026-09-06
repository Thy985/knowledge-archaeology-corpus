# 02 Engineering Knowledge — EK Graph（宽底座）

> 28 条 EK，全部来自可读实现（lib/ 无混淆）。证据均可回溯 999a0ed。

## EK-01 双轨道记忆（user track vs learned track）
- **内容**：user track（MEMORY.md/USER.md）只由显式用户动作（memory 工具调用）或用户确认的建议写入；learned track（SUGGESTIONS.jsonl）由后台审查提议、用户经 /memory_review 确认后转正。写入权威分离。
- **证据**：lib/index.js 头部注释（L9-13）
- **epistemic**: Fact
- **links**: [causal: EK-01→EK-02（轨道→审查）], [subsystem: EK-01/EK-04 同记忆面]

## EK-02 回合内记忆审查：主 LLM 自审 + 插件节拍器
- **内容**：主 LLM 审查自己的会话（持有完整上下文——无 subagent/无摘要/无转录重建）；插件只提供 pace-maker（agent/turn-stopping 计数到 reviewInterval 触发 DUE）与写路径。
- **证据**：lib/review.js 头部注释（L4-15）
- **epistemic**: Fact
- **links**: [contrast: EK-03], [causal: EK-02→EK-08（审查→学习轨道）]

## EK-03 节拍器不自动重置（防静默丢弃）
- **内容**：turn 计数器永不自动重置——只有模型 memory_review_status complete 调用才重置；错过/中断的审查在下一回合保持 DUE，不会静默消失。subagent 会话不计数。
- **证据**：lib/review.js（"The counter is never auto-reset"）
- **epistemic**: Fact
- **links**: [contrast: EK-02（自审 vs 计数）], [mechanism: EK-07（确认门）]

## EK-04 五轨记忆分层
- **内容**：用户档案（偏好/沟通习惯，每回合可见）+ 全局事实（环境/工具/惯例）+ 项目关键记忆（约定/决策/架构，自动注入，可标记 git 分支生效）+ 项目日志/每日日志（自动记录进展，按需读取）。分层决定注入成本与检索面。
- **证据**：README 场景一（L56-77）
- **epistemic**: Fact（文档 + store 实现）
- **links**: [causal: EK-04→EK-19（项目记忆→分支生效）], [subsystem: EK-01/EK-04 同记忆面]

## EK-05 Hermes 兼容纯文本条目格式
- **内容**：记忆文件为纯文本，`\n§\n` 条目分隔（byte-compatible with Hermes MEMORY.md/USER.md）；parseEntries 拆条 trim 过滤空。**零依赖、外部可读可改**。
- **证据**：lib/store.js（ENTRY_DELIMITER + parseEntries）
- **epistemic**: Fact
- **links**: [mechanism: EK-06（漂移守卫依赖此格式）]

## EK-06 漂移守卫（round-trip 保护）
- **内容**：全文件重写（replace/remove）前强制 round-trip 检查——磁盘内容必须能解析往返，否则拒绝重写（防手改/外部写入丢失）；漂移文件先备份 `<file>.bak.<ts>` 再拒绝；add 追加模式跳过守卫但拒绝"存在且读为空"的文件（防抹历史）。
- **证据**：lib/store.js 头部注释（L5-10）
- **epistemic**: Fact
- **links**: [dependency: EK-06 依赖 EK-05（可解析性）], [causal: →Candidate C-02]

## EK-07 跨进程锁 + 原子写
- **内容**：每目录一个锁文件串行化所有写（多 DSH 进程/外部编辑器不能交错写）；STALE_LOCK_MS=10s 视为废弃锁、LOCK_TIMEOUT_MS=5s 超时 loud fail、自旋 25ms；全部同步写（文件很小）。
- **证据**：lib/store.js（STALE_LOCK_MS/LOCK_TIMEOUT_MS/LOCK_RETRY_MS）
- **epistemic**: Fact
- **links**: [mechanism: EK-06（都是写安全）]

## EK-08 学习轨道：SUGGESTIONS.jsonl + 用户确认
- **内容**：suggest 模式追加 SUGGESTIONS.jsonl（learned track）；auto 模式直接写全局记忆（主会话不受门控）；用户经 /memory_review 或设置面板确认。
- **证据**：lib/review.js（"suggest mode appends to the SUGGESTIONS.jsonl queue"）
- **epistemic**: Fact
- **links**: [causal: EK-08→EK-01（确认→user track）], [mechanism: EK-02]

## EK-09 情绪反馈记录（【反馈】行）
- **内容**：用户对工作结果的评价（"太好了"/"怎么还没改对"）被记入当日与项目日志（【反馈】行，含情绪、任务分类、原话）——积累后可分析用户对哪类任务不满。
- **证据**：README 场景一（L70-76）
- **epistemic**: Fact（文档 + review 实现）
- **links**: [causal: EK-09→EK-28（反馈→学习信号）], [mechanism: EK-08]

## EK-10 skill_manage read-before-write（证据门）
- **内容**：patch 整个 SKILL.md 前 REQUIRES 会话日志含 `skill_manage action=read <name>` 事件——防 AI 编辑从未读过的技能（Hermes 保护）。读日志是唯一证据源。
- **证据**：lib/skills.js 头部注释（L9-12）
- **epistemic**: Fact
- **links**: [mechanism: EK-11（路径安全）], [causal: →Candidate C-03]

## EK-11 技能命名 kebab-case 规则化防穿越
- **内容**：create 写 `<dir>/<name>/SKILL.md`，name 必须 kebab-case——规则同时排除路径穿越；body 必须 canonical SKILL.md（frontmatter 声明 name+description）；大小上限 + 原子写。
- **证据**：lib/skills.js 头部注释（L7-8）
- **epistemic**: Fact
- **links**: [mechanism: EK-10（都是技能治理）]

## EK-12 技能禁用同步（共享运行时注册表）
- **内容**：disabled 技能（任何插件通过 core ctx.skills 注册 modelInvocable:false shadow）跳过——读共享运行时注册表而非其他插件私有状态；无 dsh-skills-manager 部署行为一致。
- **证据**：lib/skills.js 头部注释（L13-15）
- **epistemic**: Fact
- **links**: [dependency: EK-12 依赖 EK-11（技能面）]

## EK-13 会话编排 spawn（与 GUI 同构）
- **内容**：spawn 程序化创建标准 DSH 会话——与 GUI 手动打开完全同构（同系统提示词/工具/记忆快照注入/持久化，左侧列表可见可接管）；首条用户消息=完整提示词；创建后自动开跑；可选 cwd/roomId/model/provider。
- **证据**：lib/session-orch.js 头部注释
- **epistemic**: Fact
- **links**: [causal: EK-13→EK-14（spawn→wake）]

## EK-14 唤醒=替用户发消息（可审计）
- **内容**：wake = sessionId + 提示词，等价替用户给对方发消息（对方 GUI 可见全过程）；正在跑则排队；进程重启后自动 resume 再唤醒；**仅同进程会话可唤醒**（跨 dsh 实例不可）。
- **证据**：lib/session-orch.js 头部注释（边界段）
- **epistemic**: Fact
- **links**: [constraint: EK-14 约束 EK-13（边界）], [causal: →Candidate C-04]

## EK-15 独立子模块纪律（用户拍板 2026-08-08）
- **内容**：明显独立的子模块不挂在别的模块下（广播曾因此从 COI 拆出）——模块组织决策有显式记录（含日期）；ws-coord 也按此纪律独立装配（installWsCoord + 独立子开关 + 独立子存储）。
- **证据**：lib/session-orch.js 头部注释（"用户拍板纪律 2026-08-08"）；ws-coord.js 头部注释
- **epistemic**: Fact（决策记录）
- **links**: [mechanism: EK-16（模块组织）], [causal: →Candidate C-05]

## EK-16 ws-coord 软模式（先信任 AI + 硬证据）
- **内容**：检测到冲突不拦截而是警告（软模式：先登记，信任 AI 自行处理）；enforceWrite 硬拦截保留为开关位备用；**fs/observed 自动登记是硬的**（每次 write/edit 成功写入即自动记入占用集，不靠 AI 自觉）。
- **证据**：lib/coi/ws-coord.js 头部注释（L6-13）
- **epistemic**: Fact
- **links**: [contrast: EK-17（软 vs 硬）], [causal: EK-16→EK-18（登记→通知）]

## EK-17 资源锁 TTL 设计（写自动续期）
- **内容**：锁 TTL 默认 1min/上限 5min（曾默认 60min，硬拦截模式下卡别人 1 小时）；**每次 write/edit 刷新 TTL**（长任务只要在写就不断续）；不写了 1-5 分钟内自然过期；回合结束不释放 observed 锁（否则先后写入检测不到——2026-08-09 用户实测教训）。
- **证据**：lib/coi/ws-coord.js 头部注释（TTL 段）
- **epistemic**: Fact（含教训记录）
- **links**: [mechanism: EK-16], [causal: →Candidate C-06]

## EK-18 冲突定向通知
- **内容**：冲突发生时给占用方发广播（notifyConflict 子开关）；tools/pre-execute 检测（write/edit）→ 软模式放行但 post-execute 注入警告上下文（additionalContexts，模型下一轮可见）；enforceWrite 时升级 deny。
- **证据**：lib/coi/ws-coord.js 头部注释（②③段）
- **epistemic**: Fact
- **links**: [causal: EK-18→EK-24（警告→模型上下文）]

## EK-19 项目记忆按 git 分支生效
- **内容**：项目关键记忆可标记只在某 git 分支生效（gitBranch 提取）；日志自动带 [git main] 分支标记可溯源。
- **证据**：README 场景一（L62-64）；store.js（gitBranch/gitBranchList）
- **epistemic**: Fact
- **links**: [causal: EK-19→EK-20（分支→同步身份）]

## EK-20 跨协议 URL 归一化身份（双设备认亲）
- **内容**：项目身份=主仓库 git remote 归一化值（origin 优先）；https/ssh/git 协议、带凭证/不带、带/不带 .git 全部收敛到同一键 `host[:port]/path`；**设备 A https clone、设备 B ssh clone 身份必须一致**；归一化失败回退 projectHash(cwd)（12 hex，与旧目录完全兼容）。
- **证据**：lib/sync/identity.js 头部注释 + normalizeRemoteUrl 实现
- **epistemic**: Fact
- **links**: [causal: EK-20→EK-21（身份→合并）], [mechanism: EK-22]

## EK-21 三路合并器（git 冲突标记永不落盘）
- **内容**：base/ours/theirs 三份记忆文件集内存合并 + 冲突清单；**不用 git merge**；entryKey 联合索引（有身份证按 ID、无 ID 按整条文本兜底；KEY.md 与 KEY-archive.md 联合）；确定性补发 ID 先行（双设备 legacy 条目补发同一 ID 才能识别"同一条两边改"）。
- **证据**：lib/sync/merge.js 头部注释（L4-21）
- **epistemic**: Fact
- **links**: [dependency: EK-21 依赖 EK-20（身份）], [mechanism: EK-23]

## EK-22 12 行规则表合并决策
- **内容**：新增保留/双侧同新增去重/未动保留/单侧修改采用/双侧同改一致去重/双侧改不同→冲突/单侧删除生效/双侧删除/改 vs 删→冲突/location 单侧变化采用、双侧不同→冲突——规则化到可测试的决策表。
- **证据**：lib/sync/merge.js 头部注释（§6 规则表）
- **epistemic**: Fact
- **links**: [constraint: EK-22 约束 EK-21（合并行为）]

## EK-23 decideModeB 分支模式（PROVENANCE 归属判定）
- **内容**：共享仓库已有本项目分支 → shared 续接；main 属于本项目（老单项目仓库）→ legacy 用 main；main 属于其他项目 → shared-fresh 绝不碰 main；main 存在但无 PROVENANCE（归属不明）→ 保守专属分支。**不碰 main 是硬约束**（e2e 测试验证）。
- **证据**：tests/sync-review-final.test.js（682-689 测试名）；lib/sync/repo.js
- **epistemic**: Fact（测试契约）
- **links**: [causal: EK-23→EK-21（分支→合并）]

## EK-24 评审发射闸门（guard）
- **内容**：accept(note) 前：①NFKC 归一化（"Stop."/*stop*/"  STOP  " → stop）②17 个空泛短语抑制（stop/done/lgtm/nothing to add…，精确匹配归一化文本，含短语的完整 note 不受影响）③每轮一条（beginUpdate 标记周期）④归一化去重 + 升级（同 note 同级/降级抑制；nit→concern→blocker 真实升级放行）。FIFO 4096 有界历史。
- **证据**：lib/advisor/guard.js 头部注释 + DEFAULT_MAX_HISTORY
- **epistemic**: Fact
- **links**: [mechanism: EK-10（都是 AI 输出治理）], [causal: EK-24→EK-25（note→投递）]

## EK-25 评审运行时生命周期（会话级）
- **内容**：guard 状态按会话生命周期——运行时随会话创建/销毁，guard 一并新建/丢弃；reset() 暴露给 compact/表面重写（KD-5 类重置）。
- **证据**：lib/advisor/guard.js 头部注释（L19-22）
- **epistemic**: Fact
- **links**: [constraint: EK-25 约束 EK-24（状态边界）]

## EK-26 零运行时依赖 + 构建注入
- **内容**：插件包零运行时依赖（node:fs/child_process/crypto only）；前端 bundle 用 esbuild（从 DSH source checkout 解析——插件内不装 esbuild）；client.js 为 CJS factory 交给 window.__ModuleLoader__.load({id,factory})，CSS 以文本注入 <style>。
- **证据**：build.mjs 头部注释；lib/client.js（18328 行产物）
- **epistemic**: Fact
- **links**: [causal: EK-26→EK-27（构建→注入）]

## EK-27 cordis.patch.yml bundle 自动注册
- **内容**：dsh.bundle.patch 声明 host 插件行自动注册（无需手动配置）；**重复 insert 同 id 会让加载器报 duplicate loader entry id 无法启动**（README 显式警告）；loader entry name 必须与 package.json name 完全一致（08-06 起标准安装，旧 @dsh-local/ 软链机制废弃）。
- **证据**：cordis.patch.yml + README 快速开始 + build.mjs（PLUGIN_ID 动态读取）
- **epistemic**: Fact
- **links**: [constraint: EK-27 约束 EK-26（入口名）]

## EK-28 外部 AI CLI 统一派单
- **内容**：skills/{codex,grok,hermes,kimi}-cli-calling 四个技能——把外部 AI CLI（Codex/Grok/Hermes/Kimi）统一调度为"外部外援"（重活派给外部 AI 代理，主会话指挥）；与 COI 内部团队配合（"一支队伍，两种兵种"）。
- **证据**：skills/ 目录 + README 场景四/五（L130-168）
- **epistemic**: Fact
- **links**: [causal: EK-28→EK-13（派单→会话）], [mechanism: EK-29]

## EK-29 记忆同步四轨独立开关
- **内容**：全局（用户档案/每日日志/待办）跨设备需填共享记忆仓库；项目轨默认用代码仓库专属分支（不污染代码）；**没开同步的项目完全不受影响**（纯本地）。
- **证据**：README 同步场景（L79-93）
- **epistemic**: Fact
- **links**: [causal: EK-29→EK-23（开关→模式）]

## EK-30 测试环境隐式契约（实测发现）
- **内容**：798 断言需三件环境契约：①git init 默认分支 main（update.test.js 硬编码 `git push origin main`）②git user 身份配置 ③search-docs darwin 假设（mdfind 优先预期）。默认环境 48 挂 → 配置后 **3 挂全为 darwin 平台假设**。
- **证据**：本 run 实测（git symbolic-ref=master 时 48 fail；配置 main+identity 后 3 fail）
- **epistemic**: Fact（实测）
- **links**: [causal: →Candidate C-07]

---

## EK 边统计（防退化检查）

- 总 EK：30；含 links：30/30（100%）；游离：0；平均出边 ~1.6
- 全部 Fact 级（lib/ 可读无混淆——与 evolver 考古形成对比，是本包可信度高于混淆项目之处）
