# 03 · Knowledge Layer — OpenCode（Generalized Knowledge，窄尖顶）

> v3.1 规范：每个 KO 声明 `aggregation_rule`（R1-R4）+ 簇内 EK。单项目证据标注 `OpenCode-originated · Cross-project validation pending`。

## 03.0 KO 聚合规则矩阵

| KO | aggregation_rule | 簇内 EK | 解释范围扩大 |
|----|-----------------|---------|--------------|
| KO-01 | R1 机制簇 | EK-01/02/05（权限门，跨 tools/permission 独立实现） | 从"bash 要批准"扩到"harness 副作用统一授权" |
| KO-02 | R2 因果链簇 | EK-04→01→02→21（agent 循环内权限门） | 从"循环结构"扩到"循环内治理点" |
| KO-03 | R4 主题簇 | EK-10/11/13/22（LSP+diff+history） | 从"LSP 集成"扩到"编辑器智能进 harness" |
| KO-04 | R4 主题簇 | EK-12/04/24/25（SQLite+会话） | 从"持久化"扩到"harness 记忆" |
| KO-05 | R1 机制簇 | EK-03/06/07/08/26（工具/MCP/provider/sourcegraph） | 从"多工具"扩到"可插拔生态" |
| KO-06 | R3 不变量簇 | EK-14/15/17/18/19/27（测试脆弱+panic+归档+失真） | 从"测试失败"扩到"增长与成熟度解耦、事实核对纪律" |

## 03.1 KO（L3 Pattern）

### KO-01 工具副作用统一授权（harness 级权限门）
- **内容**：所有改变状态的工具（bash/edit/write/patch/fetch）统一注入 permission.Service，Run 前 Request；拒绝 = ErrorPermissionDenied；工具自身不判断权限，授权与执行解耦。
- **回溯**：EK-01/02/05。
- **认知状态**：Pattern（L3）· 与 omnigent 策略执行点、Dogwood 调用前判定同域 · Cross-project validation pending。

### KO-02 agent 循环内治理点（工具调用即授权检查点）
- **内容**：生成→工具调用→权限门→执行→回喂的循环中，权限门是每个副作用动作的强制检查点——治理不依赖模型自觉，而在执行路径上。
- **回溯**：EK-04→01→02→21。
- **认知状态**：Pattern（L3）· 与 Dogwood（策略即代码）互补：这里是运行时门，Dogwood 是声明式语言 · Cross-project validation pending。

### KO-03 编辑器智能进 harness（LSP 集成）
- **内容**：LSP protocol（生成代码）+ watcher 文件变更 + diagnostics 进上下文 + diff/history 可视化——单体 TUI harness 把"编辑器级代码智能"内建（类型错误/文件状态直接可见可引用）。
- **回溯**：EK-10/11/13/22。
- **认知状态**：Pattern（L3）· 与 DeepSeek Harness（CLI 单体无 LSP 深度）对照 · Cross-project validation pending。

### KO-04 harness 记忆（SQLite 持久化会话）
- **内容**：会话/消息/文件快照 SQLite 持久化（SQLC + goose + 内嵌驱动），会话可恢复；消息附件模型 + 标题生成——harness 的"记忆"是结构化持久层而非模型上下文。
- **回溯**：EK-12/04/24/25。
- **认知状态**：Pattern（L3）· 与 MemGraphRAG 记忆主题连接 · Cross-project validation pending。

### KO-05 可插拔生态：工具/MCP/provider 三轴
- **内容**：内置工具 + MCP 服务器工具（前缀命名）+ 多 LLM provider（含 Copilot 认证直连）+ 外部代码搜索（sourcegraph）——harness 能力面由三轴可插拔构成。
- **回溯**：EK-03/06/07/08/26。
- **认知状态**：Pattern（L3）· 与 omnigent 双评估/工具注册、DeepSeek Harness 对比 · Cross-project validation pending。

### KO-06 增长与工程成熟度解耦 + 事实核对纪律
- **内容**：早期项目（42k 行）仅 4 个测试文件、panic-instead-of-error、随后归档迁移——"增长最快"不等于"工程成熟"；候选卡数据（147k★/headless）与仓库实际（13.7k★/TUI/归档）严重不符，考古必须先核对一手事实。
- **回溯**：EK-14/15/17/18/19/27。
- **认知状态**：Pattern（L3）· 跨项目 Candidate（C-03）支撑 · Cross-project validation pending。

## 03.2 Cognitive Models（L4）

### CM-01 授权必须发生在执行路径上，而非依赖模型自觉
- **内容**：**agent 安全的关键不是提示模型"要小心"，而是在工具调用与执行之间的路径上设置强制授权点——模型可绕过一切软约束，但绕不过执行层门禁。**
- **回溯**：KO-01/02（EK-01/02/04）。
- **认知状态**：Cognitive Model（L4）· 与 Dogwood 调用前判定、omnigent 策略执行点三项目印证中。

### CM-02 编辑器智能是 harness 的差异化维度
- **内容**：**在模型能力同质化的时代，harness 的差异化来自"感知与行动的深度"——LSP 文件级感知、diff 可视化、诊断注入是单体 TUI 路线对无头 harness 的结构性优势。**
- **回溯**：KO-03（EK-10/11）。
- **认知状态**：Cognitive Model（L4）· 单项目强证据。

### CM-03 审批 UX 决定 harness 可用性（同步阻塞 vs 超时）
- **内容**：**权限审批的等待模型（同步阻塞无超时 vs 超时静默拒绝）直接决定 agent 的响应性与安全性权衡——两种设计各有失败模式（挂起 vs 误拒）。**
- **回溯**：KO-01（EK-05）。
- **认知状态**：Cognitive Model（L4）· 与 omnigent ASK（86400s 超时）形成双向对照。

### CM-04 增长的可见性 ≠ 工程健康的证据
- **内容**：**开发者生态信号（star/月活/合作）与代码工程健康（测试/错误处理/边界）是正交维度——一个可以飙升，另一个可以停滞；考古/选型都必须分开度量。**
- **回溯**：KO-06（EK-14/15/19）。
- **认知状态**：Cognitive Model（L4）· 单项目强证据。

## 03.3 Methodologies（L5）

### M-01 harness 副作用动作先过权限门，再谈功能
- **内容**：**构建 agent harness 时，先定义"哪些工具改变状态 + 统一授权接口 + 审批 UX"，再扩展工具面——授权是 harness 的第一基础设施。**
- **回溯**：KO-01/02。
- **认知状态**：Methodology（L5）· OpenCode 验证。

### M-02 考古选型先核一手事实（star/架构/归档状态）
- **内容**：**第三方候选卡/文档的"增长数据与架构描述"必须用 GitHub API + 仓库结构实测复核后才可进入考古——避免基于失真画像构建知识。**
- **回溯**：KO-06（EK-19）。
- **认知状态**：Methodology（L5）· 本 run 实证（候选卡失真被修正）。

### M-03 归档快照同样可考古（演化谱系价值）
- **内容**：**已归档仓库是设计演化的化石：早期架构选择、失败模式、迁移决策都在其中——"项目不活跃"不等于"没有考古价值"。**
- **回溯**：KO-06（EK-18）。
- **认知状态**：Methodology（L5）· OpenCode 验证。

## 03.4 三层配比

| 层 | 目标 | 实际 |
|----|------|------|
| Facts/Evidence | 100+ | ~32（01 层 + EK 证据） |
| Engineering Knowledge | 40~60 | 27（小项目按比例缩放） |
| KO（L3） | 7~12 | 6 |
| CM（L4） | 少量 | 4 |
| M（L5） | 少量 | 3 |
