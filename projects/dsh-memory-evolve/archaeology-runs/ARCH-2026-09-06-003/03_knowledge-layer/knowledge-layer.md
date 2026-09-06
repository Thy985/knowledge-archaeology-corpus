# 03 Knowledge Layer — Generalized Knowledge（窄尖顶）

## 聚合总览

| KO | 规则 | 簇内 EK | 主题 |
|----|------|---------|------|
| KO-01 | R2 因果链簇 | EK-01→02→03→08 | 双轨记忆 + 确认门审查闭环 |
| KO-02 | R1 机制簇 | EK-05/06/07 | 本地记忆文件写入安全三件套 |
| KO-03 | R2 因果链簇 | EK-20→21→22→23 | 跨设备记忆同步链 |
| KO-04 | R1 机制簇 | EK-10/11/24 | AI 输出/动作治理（证据门 + 规则化 + 闸门） |
| KO-05 | R2 因果链簇 | EK-13→14→16→17→18 | 会话协作与冲突协调链 |
| KO-06 | R4 主题簇 | EK-26/27/12/15 | 插件工程与模块纪律 |
| KO-07 | R3 不变量簇 | EK-30 + 全测试面 | 测试环境契约（干净环境全绿不变量） |
| KO-08 | R4 主题簇 | EK-09/28/29/19 | 学习与生态面 |

---

## KO-01 Pattern — 双轨记忆 + 用户确认门（审查提议 ≠ 直接写入）
- **aggregation_rule**: R2 因果链簇。EK-01→02→03→08 沿 causal 边成链。
- **内容**：记忆写入分两条权威轨：user track（显式用户动作/确认）vs learned track（后台审查建议 → 用户 /memory_review 确认转正）。回合内审查由主 LLM 自审（持有完整上下文），插件只做节拍器——且**计数器不自动重置**（memory_review_status complete 才重置，防错过审查被静默丢弃）。
- **证据**：EK-01/02/03/08
- **epistemic**: Pattern（S3/S4——测试契约 + 实现精读）
- **links**: [causal: →CM-1], [causal: →M-1]

## KO-02 Pattern — 本地文件写入安全三件套（格式兼容 + 漂移守卫 + 锁/原子写）
- **aggregation_rule**: R1 机制簇。EK-05/06/07 共享"文件持久化安全"机制，跨格式/守卫/锁三实现。
- **内容**：①纯文本 + Hermes 兼容分隔（外部可读可改）②漂移守卫：全文件重写前 round-trip 解析验证，失败先备份再拒绝（防手改/并发写丢失）③每目录跨进程锁 + 原子写 + 超时 loud fail。
- **证据**：EK-05/06/07
- **epistemic**: Pattern
- **links**: [causal: →CM-1]

## KO-03 Pattern — 跨设备记忆同步链（身份归一 → 三路合并 → 分支模式）
- **aggregation_rule**: R2 因果链簇。EK-20→21→22→23 沿 causal 边成链。
- **内容**：①跨协议 URL 归一化身份（https/ssh 收敛同键——"双设备认亲"，归一化失败回退旧哈希保持兼容）②三路合并器（git 冲突标记永不落盘，内存 12 行规则表决策，确定性补发 ID 识别双侧修改）③decideModeB 分支模式（PROVENANCE 归属判定：shared/legacy/shared-fresh/保守专属；**绝不碰 main** 是硬约束）。
- **证据**：EK-20/21/22/23
- **epistemic**: Pattern
- **links**: [causal: →M-2]

## KO-04 Pattern — AI 动作治理三件套（证据门 + 规则化 + 发射闸门）
- **aggregation_rule**: R1 机制簇。EK-10/11/24 共享"约束 AI 动作"机制，跨技能/评审两面。
- **内容**：①read-before-write（patch 前需会话日志 read 事件证据）②kebab-case 规则化（同时排除路径穿越）+ 大小上限 + 原子写 ③评审发射闸门（NFKC 归一化 + 17 空泛短语抑制 + 每轮一条 + 去重升级 nit→concern→blocker）。
- **证据**：EK-10/11/24
- **epistemic**: Pattern
- **links**: [causal: →CM-2]

## KO-05 Pattern — 会话协作与冲突协调链（spawn→wake→广播→ws-coord）
- **aggregation_rule**: R2 因果链簇。EK-13→14→16→17→18 沿 causal 边成链。
- **内容**：①spawn 标准会话（与 GUI 同构）②wake=替用户发消息（可审计，同进程边界）③广播房间 + 存在感 ④ws-coord：软模式（先信任 AI，警告不拦截）+ 硬证据（fs/observed 自动登记）+ TTL 锁（写自动续期 1-5min）+ 冲突定向通知（post-execute 注入警告上下文）。
- **证据**：EK-13/14/16/17/18
- **epistemic**: Pattern
- **links**: [causal: →CM-3]

## KO-06 Pattern — 插件工程纪律（零依赖 + 注入机制 + 独立子模块）
- **aggregation_rule**: R4 主题簇。EK-26/27/12/15 围绕"插件如何被构建/装配/组织"覆盖互补维度。
- **内容**：①零运行时依赖（node:fs only；构建期 esbuild 从宿主 checkout 解析）②cordis.patch.yml bundle 自动注册（重复 insert 崩溃警告 + loader name 必须等于 package name）③独立子模块纪律（用户拍板 2026-08-08：独立模块不挂别处，防"拆不开"）④共享运行时注册表读取（不碰插件私有状态）。
- **证据**：EK-26/27/12/15
- **epistemic**: Pattern
- **links**: [causal: →M-3]

## KO-07 Cognitive Model — 测试环境契约不变量（干净环境全绿）
- **aggregation_rule**: R3 不变量簇。EK-30 + 全部测试面汇聚到"测试在正确环境契约下全绿"这一不变量。
- **内容**：798 断言在默认环境 48 挂、配置 git 契约（main 分支 + 身份）后仅剩 3 挂（全为 darwin 平台假设）——**"测试失败"必须先区分环境契约 vs 真实回归**，否则误报回归。
- **证据**：本 run 实测（EK-30）
- **epistemic**: Cognitive Model（本项目验证 + 与 evolver 考古 2/60 env-fail 同型）
- **links**: [causal: →M-4]

## KO-08 Pattern — 学习与生态面（情绪反馈 + 外部派单 + 同步扩展）
- **aggregation_rule**: R4 主题簇。EK-09/28/29/19 围绕"记忆如何学习/扩展"覆盖互补维度。
- **内容**：①情绪反馈记录（【反馈】行含情绪/任务分类/原话——可分析用户不满的任务类型）②外部 AI CLI 派单（codex/grok/hermes/kimi 统一调度）③四轨同步独立开关（未开项目纯本地不受影响）④项目记忆按 git 分支生效。
- **证据**：EK-09/28/29/19
- **epistemic**: Pattern
- **links**: [causal: →CM-1]

---

## L4 认知模型（Cognitive Model）

### CM-1 AI 记忆可信性 = 用户确认 + 可审计轨道 + 漂移防护
**一句话**：让 AI 拥有"可信的长期记忆"需要三层：写入权威归用户（确认门）、轨道可审计（user/learned 分离）、文件防漂移（round-trip 守卫）——记忆系统的信任不是来自 LLM 的可靠性，而是来自写入管道的纪律。
- 支撑：KO-01/KO-02/KO-08。本项目验证（S3/S4）。与 evolver 的"GEP 资产不覆盖承诺"同型（都是资产保护）。

### CM-2 智能 ≠ 权威（AI 的"会"不等于 AI 的"能改"）
**一句话**：AI 工具对持久状态的每次修改都需要证据门（read-before-write）与规则化约束（kebab-case）——LLM 的能力（智能）与修改权限（权威）必须分离治理。
- 支撑：KO-04。与 RAMPART/evolver 的"intelligence ≠ authority"跨项目呼应（第三个验证点）。

### CM-3 协作冲突管理 = 先信任 + 硬证据 + 短 TTL
**一句话**：多 AI 会话协作的冲突管理应默认软约束（信任 AI 自行处理），但自动登记（fs/observed）是硬的、锁 TTL 要短且随写入续期——"信任"与"证据"分层，不因信任放弃可检测性。
- 支撑：KO-05。本项目验证。

### CM-4 插件生态的零侵入是可持续性前提
**一句话**：宿主系统插件若只走 public seams（systemPrompt/tools/commands/subagents/approval）、零运行时依赖、构建期借用宿主工具链，则插件可随宿主演进而不碎——侵入越少，生存越久。
- 支撑：KO-06。本项目验证。

---

## L5 方法论（Methodology）

### M-1 记忆系统写入纪律
可操作准则：①写入口分权威轨（用户确认 vs 建议待确认）②审查节拍器不自动重置（防静默丢弃）③文件格式选外部可读纯文本 + 分隔符④全文件重写前 round-trip 验证 ⑤每目录锁 + 原子写。

### M-2 跨设备同步设计
可操作准则：①身份用远端 URL 归一化（协议收敛）而非路径②合并器内存决策、冲突标记不落盘③归属不明的分支保守化（不碰 main）④规则表（12 行）化到可测试。

### M-3 插件工程三原则
可操作准则：①零运行时依赖（node:fs only）②bundle patch 自动注册 + 入口名=包名硬约束③独立子模块独立装配（防"拆不开"）。

### M-4 测试失败先分环境契约
可操作准则：任何测试套件全量失败先检查环境契约（git 默认分支/身份/平台假设/外部服务）再判回归——环境契约是测试的隐式前置条件，须文档化。

---

## 升维纪律检查

| 检查 | 结果 |
|------|------|
| 每条 KO 有 aggregation_rule | ✅ 8/8 |
| 无"同子系统=聚合理由"假聚合 | ✅ |
| 每个 L4 有支撑 KO | ✅ CM-1~4 |
| Cross-project 标注 | ✅（CM-2 三个验证点；CM-4 与 evolver 同型） |
| Hypothesis 未冒充 Fact | ✅（C-01~07 在 05） |
| 证据强度 | ✅ 全包 S3/S4（lib/ 可读 + 测试实测） |
