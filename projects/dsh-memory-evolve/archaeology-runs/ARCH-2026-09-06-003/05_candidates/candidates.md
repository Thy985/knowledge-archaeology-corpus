# 05 Candidates — 未确认内容

## C-01 [Cross-project Hypothesis] 节拍器不自动重置是否是可复用审查纪律
- **内容**：review 计数器只在模型 complete 调用时重置——错过/中断的审查保持 DUE。此纪律防静默丢弃，但代价是审查可能永久 DUE（模型从不 complete）。是否成为通用模式（周期性自我审查系统的节拍器设计）待跨项目验证。
- **证据**：lib/review.js（never auto-reset）
- **epistemic**: Hypothesis（本项目实现；通用性 pending）
- **scope**: Cross-project

## C-02 [Tentative Finding] 漂移守卫的 add 例外缺口
- **内容**：add 追加模式跳过漂移守卫（append-only 语义正确），但拒绝存在且读为空的文件（防抹历史）。边界：追加模式不验证磁盘现有内容可解析——若文件已被外部破坏（不可解析），add 仍追加（守卫不触发），只有 replace/remove 才触发守卫。这是设计权衡（append 不该管历史）还是缺口待确认。
- **证据**：lib/store.js 头部注释（L6-9）
- **epistemic**: Observation（边界语义需人工确认）

## C-03 [Scope-uncertain] read-before-write 的读证据可伪造性
- **内容**：patch 前检查会话日志 tool/call 事件（skill_manage action=read）。若 AI 在同一会话先 read 后 patch，证据链成立；但 read 证据只证明调用过 read 工具，不证明确实读取并理解内容——证据强度边界需确认。
- **证据**：lib/skills.js（REQUIRES prior read evidence）
- **epistemic**: Hypothesis（证据语义边界）

## C-04 [Tentative Finding] 唤醒边界：同进程限制
- **内容**：wake 仅同进程会话可唤醒（跨 dsh 实例/跨机器不可）——分布式会话编排是明确不支持的边界。跨机器唤醒是否作为未来方向（与 sync 的跨设备配合）未确认。
- **证据**：lib/session-orch.js（仅同进程会话可唤醒）
- **epistemic**: Observation（明确边界记录）

## C-05 [Observation] 用户拍板纪律内嵌代码的工程价值
- **内容**：模块头部注释含用户拍板（2026-08-08 独立子模块 / 2026-08-09 锁 TTL / 2026-08-10 身份方案 v13）——设计决策以日期+理由内嵌代码。此类决策注释是否优于外部 ADR（可追溯性/可检索性）待比较验证。
- **证据**：session-orch.js/ws-coord.js/identity.js 头部注释
- **epistemic**: Observation（模式价值 pending）

## C-06 [Tentative Pattern] 冲突锁 TTL 随写入活动续期
- **内容**：锁 TTL 1-5min + 每次 write/edit 刷新——正在干活的临时信号随活动续期、不写自然过期。此设计对任何协作资源锁（CI 构建锁/文档锁）有通用价值；曾默认 60min 卡人 1 小时是失败教训。
- **证据**：ws-coord.js（TTL 设计 + 教训注释）
- **epistemic**: Pattern（本项目）/ Cross-project pending

## C-07 [Tentative Finding] 测试环境契约未文档化
- **内容**：测试套件隐含三件环境契约（git main 默认分支/身份/darwin 假设），README/测试注释未集中声明——新环境 48 挂会误判回归。建议：环境契约文档化 + CI 固定（init.defaultBranch=main + 身份注入 + darwin-only 测试 skip 标记）。
- **证据**：本 run 实测（48 挂 → 3 挂）
- **epistemic**: Observation（本 run 实测）；改进建议 pending

## C-08 [Cross-project Hypothesis] DSH 生态插件与 Evolver 派单的生态重叠
- **内容**：本项目 skills/ 含 hermes-cli-calling（派单给 Hermes/EvoMap 生态）；evolver 考古发现 README 声明与 hermes-agent（OpenClaw）相似性争议；KnowlegeMap 雷达含 hermes-agent。三个生态节点（DSH/evolver/hermes）围绕外部 AI 调度汇合——跨项目外部 AI 统一调度模式是否成立待验证。
- **证据**：skills/hermes-cli-calling；evolver README Notice；KnowlegeMap radar
- **epistemic**: Hypothesis（跨项目）
- **scope**: Cross-project

---

## 候选处置

| 候选 | 处置 |
|------|------|
| C-01/C-08 | 保持 Hypothesis（跨项目待验证） |
| C-02/C-06 | 保持 Tentative（边界/模式） |
| C-03 | 保持 Hypothesis（证据语义） |
| C-04/C-05 | 保持 Observation（明确边界/模式价值） |
| C-07 | 保持 Observation + 改进建议（环境契约文档化） |
