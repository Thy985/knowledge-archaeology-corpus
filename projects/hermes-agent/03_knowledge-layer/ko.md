# Knowledge Layer — Generalized KO（hermes-agent @ fba4cb1）

> 每个 KO 声明 aggregation_rule（R1-R4 之一）+ 簇内 EK + 边类型 + 解释范围扩大论证。
> L1→L5 严格区分：Fact / Observation / Hypothesis / Pattern / Cognitive Model / Principle。

## KO-01 记忆冻结契约（L3 Pattern）
- **声明**：长生命周期 agent 的持久记忆应以"会话边界冻结快照"进入系统提示词，会话内写盘不改变 prompt——用 session 粒度的一致性换 prefix cache 复用。
- **aggregation_rule**: R2 因果链簇 —— EK-01→EK-02→EK-03→EK-22（cache 神圣性 → 冻结设计 → deferred 变更纪律 → 唯一例外压缩）
- **簇内 EK**: EK-01, EK-02, EK-03, EK-22, EK-40（causal 链 + 旁路面）
- **解释范围扩大**：同一"记忆进 prompt"问题在 letta（投影已提交内容）、Claude Code（记忆文件）均出现"只读已提交/冻结"模式 → 跨实现成立
- **Epistemic**: Validated Pattern（本 run F-03/F-22 + letta run 交叉对照，S4/S8 混合）
- **反例**：mid-session 写盘立即需生效的场景（紧急指令）不适用——hermes 用 `--now` opt-in 覆盖；**skip_memory=True 时冻结加载被跳过（旁路面，R-1）**

## KO-02 唯一承重边界原则（L4 Cognitive Model）
- **声明**：面对对抗性 LLM 输入，in-process 启发式防御（注入扫描/文件检查）永远不是承重边界；唯一承重边界是执行域隔离（OS 级/沙箱级）。
- **aggregation_rule**: R3 不变量簇 —— EK-22（唯一边界声明）约束 EK-23（file_safety 非边界）/EK-24/EK-25（注入扫描是协作性防御）/EK-26
- **簇内 EK**: EK-22, EK-23, EK-24, EK-25, EK-26（constraint 边汇聚）
- **解释范围扩大**：letta 以 seatbelt/bwrap 内核沙箱为承重墙（fail-closed）；hermes 明文 OS 隔离为唯一边界——两个独立实现得出同构认知：**判定力（heuristics）≠ 隔离力（enforcement）**，安全强度由最强物理隔离决定，不由防御层数量决定
- **Epistemic**: Cognitive Model（双项目同构，Cross-project validation pending→S8 方向）
- **反例**：威胁模型=完全可信模型（无对抗性输入）时，协作性拒绝即够——hermes SECURITY.md 明确单租户场景下该假设成立

## KO-03 记忆的窄腰设计（L3 Pattern）
- **声明**：记忆系统应做窄接口（单一 memory 工具 + 单外部 provider），避免多 provider 并行造成的 schema 膨胀与后端冲突。
- **aggregation_rule**: R1 机制簇 —— EK-05/EK-06/EK-26 共享"单一入口约束"机制，跨 memory 子系统与工具系统
- **簇内 EK**: EK-05, EK-06, EK-26（mechanism 边）
- **解释范围扩大**：与 AGENTS.md"core is a narrow waist"同构——工具面窄、记忆面窄、核心面窄，是同一设计哲学的三处实例化
- **Epistemic**: Pattern（本 run S4 测试支撑 + 架构明文）
- **反例**：多 provider 并行的记忆聚合场景（letta 支持多后端配置）——hermes 为此显式付出"只能单外部"约束

## KO-04 供应链信任分级（L3 Pattern）
- **声明**：外部技能/插件进入 agent 前必须按来源信任分级 + 静态扫描；agent 自建内容的危险模式应"ask 重试"而非静默放行。
- **aggregation_rule**: R4 主题簇 —— EK-13/EK-14/EK-11/EK-12 覆盖供应链安全的信任分级/扫描/硬验证/lint 四维
- **簇内 EK**: EK-13, EK-14, EK-11, EK-12（主题互补）
- **解释范围扩大**：OWASP ASI04/Skills Top10 同线；SkillFortify（corpus）形式化验证 vs hermes 静态扫描 vs letta frontmatter 门禁——供应链防线从"格式"到"信任"到"证明"三层光谱
- **Epistemic**: Pattern（本 run S3/S4 + 行业规范对照）
- **反例**：--force 覆盖路径存在（community caution 可强装）——显式逃生舱

## KO-05 状态韧性工程族（L3 Pattern）
- **声明**：持久状态库需要系统性韧性族：WAL 兼容回退、生产/测试实例隔离、回滚权威模型、文件级检查点——不是单点 try/except。
- **aggregation_rule**: R4 主题簇 —— EK-17/EK-18/EK-19/EK-20/EK-21 覆盖状态存储的持久化/兼容/隔离/回滚/快照
- **簇内 EK**: EK-17, EK-18, EK-19, EK-20, EK-21
- **解释范围扩大**：SQLite WAL 兼容问题（NFS/ZFS/FUSE）是跨项目普适工程知识；"durable transcript 为权威"模型可迁移到任何 undo 系统
- **Epistemic**: Pattern（本 run S3/S4 测试强支撑）
- **反例**：只读场景无需回滚权威模型——rewind 仅对可编辑会话激活

## KO-06 自我改进闭环（L4 Cognitive Model）
- **声明**：agent 的成长 = 声明性记忆（MEMORY.md/USER.md）+ 程序性记忆（SKILL.md，agent 通过 skill_manage 自主创建/编辑）+ 生命周期维护（Curator 调度器）；**改进（agent 意图驱动）与维护（调度器策略驱动）分离**；改进动作必须可恢复（只归档不删除）且不干扰主会话（auxiliary fork）。
- **aggregation_rule**: R2 因果链簇 —— EK-09→EK-10→EK-11→EK-15→EK-16（技能即记忆→布局→治理→空闲维护→不变量）+ EK-40（旁路面）
- **簇内 EK**: EK-09, EK-10, EK-11, EK-15, EK-16, EK-40
- **解释范围扩大**：letta 的 memory-subagent 家族与 hermes 的 skill_manage 在"改进"上同构（agent 主动写记忆/技能）；hermes 的 Curator 在"维护"上引入调度器自动化——**改进与维护分离**是成长型 agent 的稳定结构（Reconciliation R-2）
- **Epistemic**: Cognitive Model（本 run S3/S4；Cross-project 与 letta 对照 S8 方向）
- **反例**：skip_memory 旁路（ID-1）；无空闲窗口场景 Curator 永不触发——设计上接受

## KO-07 上下文经济学（L4 Cognitive Model）
- **声明**：上下文窗口是 agent 的第一稀缺资源；prompt cache、轨迹压缩（保护 head+尾、不拆工具对）、冻结快照、deferred 变更——全部是同一"上下文预算"约束下的不同策略。
- **aggregation_rule**: R1 机制簇 —— EK-02/EK-33/EK-01/EK-03 共享"token 预算管理"机制，跨 prompt 构建与轨迹后处理
- **簇内 EK**: EK-02, EK-33, EK-01, EK-03
- **解释范围扩大**：letta 的 post-turn push + compression、Claude Code 的 compact——上下文经济学是 agent harness 普适主线
- **Epistemic**: Cognitive Model（本 run + letta run 交叉，S4/S8）
- **反例**：无 cache 的 stateless 调用场景（单次 batch）策略退化——batch_runner 独立于该模型

## KO-08 人格即配置（L2 Observation）
- **声明**：SOUL.md 作为人格/风格文件与 config.yaml 同信任类；人格文件进入注入扫描范围但 user_authored 例外。
- **aggregation_rule**: R4 主题簇（小型）—— EK-34/EK-25/EK-26 覆盖人格文件信任类/扫描策略/写保护
- **簇内 EK**: EK-34, EK-25, EK-26
- **Epistemic**: Observation（本 run S3；属项目局部观察，不升 L3——人格文件设计无跨项目普适性证据）
- **反例**：distribution 分发的人格文件（user_authored=False）被 block——信任类差异生效

## KO-09 多前端单核心（L2 Observation）
- **声明**：同一 agent 核心通过薄适配层服务 CLI/gateway/TUI/desktop 多前端；扩展靠 plugins+skills。
- **aggregation_rule**: R4 主题簇 —— EK-30/EK-31/EK-29 覆盖多前端/入口纪律/注册发现
- **簇内 EK**: EK-30, EK-31, EK-29
- **Epistemic**: Observation（本 run S3；产品架构事实，知识价值中等，不升维）

## 知识层统计
- KO 总数：9（L3×4 / L4×3 / L2×2）
- 聚合规则：R1×2 / R2×2 / R3×1 / R4×4（覆盖率 100%）
- 反例：≥2/KO（总计 20+ 定向反例，见 06_validation）
