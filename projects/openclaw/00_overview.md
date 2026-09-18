# 00 Overview — OpenClaw 2026.9.4（ARCH-2026-09-17-001）

## 定位一句话
OpenClaw 是一个本地优先的开源 AI 助手 Gateway（MIT，Node.js/TypeScript monorepo，43,727 文件/821MB），把 Claude/Codex/本地模型等 harness 做成可插拔插件，接入 20+ 聊天渠道与全平台客户端；本次考古聚焦 **2026.9.2 权限模型变更**（swarm 默认开启 + 跨 agent 会话访问默认扩大）——这是用户 campus_order 升级 OpenClaw 前的强制审查项（KnowlegeMap 09-17 扫描日志"下一步"第一优先级）。

## 关键数字
| 项 | 值 |
|---|---|
| version | 2026.9.4（HEAD ca38979d，2026-09-16） |
| 变更焦点版本 | 2026.9.2（2026-09-05 发布，1,245 PRs + 232 contributors） |
| 规模 | 43,727 文件 / 821MB / 35,463 .ts |
| 测试 | 15,643 *.test.ts + 864 e2e（未全量本地运行，仓库规模超本地执行预算） |
| packages | 22 个（acp-core/agent-core/llm-core/net-policy/sdk/…） |
| apps | 8 个客户端（android/ios/macos/linux/mobile/shared/…） |

## 5 Top Findings
1. **默认值漂移是安全变更的载体**：2026.9.2 中 `tools.sessions.visibility` 省略值从 `agent` 改为 `all`、`tools.agentToAgent.enabled` 省略值从 `false` 改为 `true`——**升级即扩权**，官方升级说明明确要求共享 Gateway 用户在升级前审查会话访问（docs/releases/2026.9.2.md:11 "Upgrade note for shared Gateways"）。这不是新功能，是默认权限面扩大。
2. **信任模型明示"会话可见性不是安全边界"**：SECURITY.md 原话 "Anyone who can operate an agent can make it do anything that agent can do. Session ownership, visibility, and presence are usability features, not security boundaries"——真实隔离的唯一手段是**独立 Gateway/信任边界**。这是与 09-14 RSAC"coding agent 100% 可注入"直接对撞的立场。
3. **安全审计是"暴露层"不是"门禁"**：`openclaw security audit` 对默认跨 agent 会话访问只报 **info/warn**（`security.trust_model.cross_agent_session_access_default`），不 fail——审计把决策留给 operator，不替用户做安全判定。
4. **swarm 默认开启带显式容量治理**：`tools.swarm.enabled=true` 默认，但 maxConcurrent=8 / maxChildrenPerGroup=50 / maxTotalPerGroup=200 / waitTimeout=600s 全部有界（swarm-config.ts DEFAULT_SWARM_CONFIG），且保留 opt-out、工具限制、Code Mode 独立开关（CHANGELOG/2026.9.2.md "Swarm is enabled by default"）。
5. **四层权限面叠加，收紧手段明确**：tool policy → session visibility（self/tree/agent/all）→ agentToAgent（enabled/allow glob）→ sandbox clamp；官方 remediation 路径完整（docs/releases/2026.9.2.md:3247）。

## 边界
- 本考古基于 2026.9.4 快照（HEAD ca38979d），权限模型叙述以 2026.9.2 changelog/docs 为准（9.4 未回退该变更）。
- 未全量运行 15,643 测试（本地执行预算）；测试揭示行为来自关键测试文件精读（audit-cross-agent-session-access.test.ts 表驱动 7+4 例）。
- 不评估"是否应该升级"——只提供权限模型变更的事实、语义与收紧路径，供 campus_order 决策。
