# 03 Knowledge Layer — Generalized Knowledge（窄尖顶）

> 从 EK Graph 按 R1-R4 聚簇。未跨项目验证标注 pending。

## 聚合总览

| KO | 规则 | 簇内 EK | 主题 |
|----|------|---------|------|
| KO-01 | R2 因果链簇 | EK-01→02→03→05 | 协议约束自进化闭环 |
| KO-02 | R1 机制簇 | EK-06/07/13/19 | 安全纵深与治理 |
| KO-03 | R2 因果链簇 | EK-09→10→12→11 | 自我更新的可恢复性 |
| KO-04 | R4 主题簇 | EK-08/15/16/17/18 | 资产与协议治理 |
| KO-05 | R4 主题簇 | EK-20/21/22/28 | 多宿主可观测面 |
| KO-06 | R4 主题簇 | EK-14/30/24 | 测量与韧性 |
| KO-07 | R4 主题簇 | EK-25/26/27/29 | 自指与开源治理 |

---

## KO-01 Pattern — 协议约束的自进化闭环（信号→选择→生成→验证→学习→回填）
- **aggregation_rule**: R2 因果链簇。EK-01→02→03→05 沿 causal 边成链。
- **内容**：自进化循环六步闭环：扫描信号 → 匹配基因/胶囊（不 improvisation）→ 生成协议约束 prompt → solidify 验证补丁 → 失败分类（soft/hard）→ 回滚（stash 优先）→ 学习信号回填基因（下次更好）。**进化是受约束的**：验证命令白名单、约束破坏硬失败、回滚可恢复。
- **证据**：EK-01/02/03/05（README + 测试契约 + CLI）
- **epistemic**: Pattern（本项目 S3/S4——测试可执行契约验证）
- **links**: [causal: →CM-1], [causal: →M-1]

## KO-02 Pattern — 安全纵深分层（白名单 + 深度防御 + 源码级回归）
- **aggregation_rule**: R1 机制簇。EK-06/07/13/19 共享"多层安全闸"机制，跨 validator/测试/资产入口 3 面。
- **内容**：四层安全：①可执行白名单（node-only，npm/npx 因 GHSA 移除）②纵深 flag 阻止 + shell 元字符拒绝 ③路径遏制（--out= cwd 逃逸拒绝）④外部资产隔离（ingest 暂存 + promote 审计）。配套：**源码级回归测试**（grep index.js 模式锁定 GHSA 修复）。
- **证据**：EK-06/07/13/19
- **epistemic**: Pattern
- **links**: [causal: →CM-2]

## KO-03 Pattern — 自我更新的可恢复性（备份 + 日志 + 原子重命名 + 金丝雀 + 崩溃恢复）
- **aggregation_rule**: R2 因果链簇。EK-09→10→12→11 沿 causal 边成链。
- **内容**：升级路径六重保障：结构化失败分类（不吞错误）→ 备份（.evolver-force-update-backup-*）→ 日志（journal）→ 安装标记验证（防误升级第三方仓库）→ 金丝雀（重启前验坏代码）→ bootstrap 崩溃恢复（fail-closed 拒绝启动而非带伤运行）。
- **证据**：EK-09/10/12/11
- **epistemic**: Pattern
- **links**: [causal: →CM-3]

## KO-04 Pattern — 资产与协议治理（本地资产不覆盖 + 协议 SDK 化）
- **aggregation_rule**: R4 主题簇。EK-08/15/16/17/18 围绕"资产与协议"覆盖互补维度。
- **内容**：①用户运行时资产（genes/capsules/events）git ignored 且升级永不覆盖（bundled 种子只首跑复制）②协议枚举进独立 SDK（防手维护漂移——v1.80.8 explore 事故教训）③Hub 同步 + .gepx 便携备份（已购资产零成本重取）。
- **证据**：EK-08/15/16/17/18
- **epistemic**: Pattern
- **links**: [causal: →M-3]

## KO-05 Pattern — 多宿主可观测面（适配器 + 代理 + trace + webui）
- **aggregation_rule**: R4 主题簇。EK-20/21/22/28 围绕"接入与可观测"覆盖互补维度。
- **内容**：统一代理面（多模型路由）+ trace/usage 提取 + 宿主会话钩子（sessions_spawn 文本协议）+ 本地 webui 观察面板——接入多宿主的同时保持可观测。
- **证据**：EK-20/21/22/28
- **epistemic**: Pattern
- **links**: [causal: →CM-3（可观测性→信任）]

## KO-06 Pattern — 测量与韧性（golden-vectors + 韧性测试轮）
- **aggregation_rule**: R4 主题簇。EK-14/30/24 围绕"验证系统自身"覆盖互补维度。
- **内容**：①一致性基准（conformance golden-vectors：savings 公式精确匹配）②韧性测试轮（heartbeatResilience Round3-9 连续 7 轮系统化）③进化记忆（memoryGraph/narrative）——系统不仅测功能，还测自身在恶劣环境下的存活与收益。
- **证据**：EK-14/30/24
- **epistemic**: Pattern
- **links**: [causal: →M-2]

## KO-07 Pattern — 自指系统与开源治理（自我修改 + 混淆 + skill 自描述）
- **aggregation_rule**: R4 主题簇。EK-25/26/27/29 围绕"系统与自身的关系"覆盖互补维度。
- **内容**：Evolver 是自指系统：能修改自身（EVOLVE_ALLOW_SELF_MODIFY 默认 false 显式治理）、自带 SKILL.md 自我描述（capability-evolver）、把 skill 转化为自身资产（skill2gep）、核心逻辑混淆 + source-available 转向（GPL-3.0 声称 vs 可审计性现实）。
- **证据**：EK-25/26/27/29
- **epistemic**: Pattern（本项目暴露）
- **links**: [causal: →CM-4], [causal: →M-4]

---

## L4 认知模型（Cognitive Model）

### CM-1 自进化的可信性 = 约束 + 可审计 + 可回滚
**一句话**：让系统改进自身的唯一安全方式是"协议约束进化 + 资产可审计 + 失败可回滚"——无约束的自我修改是事故，不是进化。
- 支撑：KO-01/KO-03。本项目验证（S3/S4）。与 RAMPART 的"fail-closed"、skillfortify 的"over-declaration 防护"形成跨项目呼应。

### CM-2 安全是"层"不是"点"（纵深防御 + 源码级回归锚定）
**一句话**：安全机制必须多层叠加（白名单/flag 阻止/路径遏制/资产隔离），且每层修复要以源码级回归锁定（防重构静默回退）。
- 支撑：KO-02。本项目验证。

### CM-3 可恢复性先于正确性（自我更新引擎的秩序）
**一句话**：对会自我更新的系统，恢复能力（备份/日志/金丝雀/崩溃恢复）的优先级高于单次更新的正确性——"不带着坏状态继续运行"（fail-closed）比"更新成功"更重要。
- 支撑：KO-03/KO-05。本项目验证。

### CM-4 开源的边界由"可审计性"而非"许可证文本"定义
**一句话**：GPL-3.0 许可证声称 ≠ 外部可审计性；混淆的"source-available"在实质上收窄了第三方验证能力——考古/评估此类项目必须以实际可读面为准。
- 支撑：KO-07。本项目暴露（与 MemGraphRAG"Multi-Agent 名实差距"同型：**声明 vs 实现现实**）。

---

## L5 方法论（Methodology）

### M-1 协议约束进化法
可操作准则：构建自进化系统时：①进化动作必须匹配既有资产（禁止 improvisation）②生成物为"协议 prompt"而非直接改码 ③验证-失败分类-回滚闭环 ④学习信号结构化回填资产。

### M-2 自进化引擎的测试四件套
可操作准则：①安全回归用源码级锁定（grep 模式，防重构回退）②一致性用 golden-vectors（公式精确匹配）③韧性用连续轮次（Round3-9 系统化）④协议用 SDK 提取（防枚举漂移）。

### M-3 资产治理三原则
可操作准则：①用户资产与系统资产分离（git ignored + 升级不覆盖）②协议枚举进独立 SDK 单一来源 ③资产可同步可备份（.gepx 便携包 + 已购零成本重取）。

### M-4 开源项目评估的名实核对
可操作准则：评估任何"开源"项目时先实测：①核心逻辑是否可读（混淆面）②许可证声称与实现一致否（doc-impl）③自我修改边界是否显式治理——三者决定该项目的真实可审计性。

---

## 升维纪律检查

| 检查 | 结果 |
|------|------|
| 每条 KO 有 aggregation_rule | ✅ 7/7 |
| 无"同子系统=聚合理由"假聚合 | ✅ |
| 每个 L4 有支撑 KO | ✅ CM-1~4 |
| Cross-project 标注 pending | ✅ |
| Hypothesis 未冒充 Fact | ✅（C-01~C-08 在 05） |
| 混淆盲区未升维 | ✅（🔒 相关 EK 仅 Fact/Observation 级） |
