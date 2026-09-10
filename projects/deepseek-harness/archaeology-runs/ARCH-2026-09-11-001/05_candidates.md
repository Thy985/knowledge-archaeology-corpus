# 05 — Candidates（未验证 / 跨项目 / 暂定）

> 规则：不能确认的内容留在 Candidate；Hypothesis 不冒充 Fact；Cross-project Candidate 不写成已验证 Principle。基线 9 个 Candidate 见 09-03 包；本节新增 5 个。

## 5.1 新增 Candidates

### C-R01 — 【Hypothesis】"决策笔记密度 ≈ 系统治理成熟度"（跨项目假说）
- 观察：dsh 在 09-05 后 notes 增速翻倍（+96 条集中在 feedback/sidebar/workspace-files/preview），同期功能面快速扩大；09-03 考古因不读 notes 而漏掉整块决策证据。
- 假说：Agent 类项目若有显式 ADR 体系（Problem→Decision→Alternatives），其"为什么"类知识可提取率显著高于仅读代码。
- 缺失证据：仅 dsh 单项目；需对第二个有 ADR 体系的项目（如 knowledge-archaeology-skill 自身）验证。
- 验证路径：对 skill-repo 做同法考古，比较 EK 中"决策解释"占比。

### C-R02 — 【Hypothesis】PTC collapse 位置（executor 内、策略前）是通用安全模式
- 观察：dsh 把折叠判定放在 `createExecution`（策略管线入口之前），使确定性拒绝永不进入审批。
- 假说：Agent harness 中"不可见工具=不可执行"的不变量，最佳实现位点是**执行边界**而非策略管线（策略只处理真实候选）。
- 缺失证据：单项目（dsh）；E2B/OpenCode 等未验证同构。
- 反例风险：若未来引入"审批后可见性提升"（approval 解锁工具），折叠点需与解锁点协调——未处理。

### C-R03 — 【Tentative Pattern】"读授权继承后端，导航边界收敛 workspace"（权限分层法）
- 观察：dsh 对具名读取 vs 导航观察采用不同边界（EK-R05）。
- 暂定模式：文件服务类功能应区分"资源读取"与"空间导航"两类操作语义，前者继承执行环境授权，后者收敛到空间边界。
- 缺失证据：仅 dsh 一例；需要第二个实现（IDE 文件树、云端 workspace 服务）对照。

### C-R04 — 【Hypothesis】iframe opaque-origin + 允许网络 = 预览类功能的安全平衡点
- 观察：dsh Document Preview 用 `sandbox="allow-scripts"` opaque origin，保留浏览器网络，明确记录为 intentional trade-off（EK-R08）。
- 假说：静态 HTML 预览的安全基线是"opaque origin 防父访问 + 保留网络"，而非禁网 CSP。
- 缺失证据：dsh 单项目；浏览器侧风险（opaque origin 泄密向量）未做安全审计（SAFETY.md 自述未经审计）。
- **需要人工复核**：此权衡是否适用于生产环境（SAFETY.md 声明非生产就绪）。

### C-R05 — 【Scope-uncertain】"181 插件生态"数字不可核验
- 雷达/媒体称 181 插件；仓库内为 everything-is-a-plugin + 55 个 packages（无 181 清单）。
- 判定：55 packages 已查证；181 = 一方称。若 181 指 npm 生态安装量/registry 包数，需另行核验。
- 建议：以仓库 packages 数（55）为准，181 标注为媒体声明。

## 5.2 基线 Candidate 存续性

- 09-03 的 9 个 Candidate（含跨项目假说 KO-03/05 关联项）**保留**；本轮证据未推翻，也未升级（除 09-03 已登记的 compaction 有损等矛盾项保持开放）。
