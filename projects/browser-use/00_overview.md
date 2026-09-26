# 00 · Overview — browser-use (ARCH-2026-09-22-001)

## 一句话定位
browser-use 是**开源事实标准的浏览器 Agent 库**（115,743★，Python ≥3.11）：Agent 通过 LLM 决策 + CDP 控制 Chromium，把"用户定义的任务"变成"网页上的动作序列"，直到任务完成。同时是**三层商业漏斗**的 OSS 顶端（OSS 库 → Browser Use Cloud 浏览器 → Hosted Agent API）。

## 本次考古回答的核心问题
"让 LLM 可靠地操作真实网页"需要哪些工程机制？本仓库给出了一个 115k★ 的工业级答案：**受控 Agent 循环（观察→决策→执行→验证）+ 事件驱动 watchdog 安全/状态守护 + 上下文预算分级降级 + 软循环检测 + 独立 judge**。

## 核心数字（事实）
- 仓库：browser-use 0.13.10，main @ `d8110c5`（2026-09-18 push），66,541 行 Python（browser_use 包），35,131 行测试
- 依赖：pydantic v2 / aiohttp / httpx / cdp-use（类型化 CDP）/ mcp / posthog（遥测）/ browser-use-sdk（云）/ browser-harness（CLI）/ 12+ LLM 供应商
- 架构：event-driven（bubus EventBus）+ 16 个 watchdog + Agent 异步主循环
- 商业模式（AGENTS.md/README 明文）：OSS 库 → `use_cloud=True` 云端浏览器（$0.02/browser-hour，stealth + CAPTCHA 解决 + 住宅代理）→ Hosted Agent API → 自有模型 ChatBrowserUse 推荐

## 知识贡献（对用户知识库）
1. **Web Agent 主循环的完整工程**：4 阶段 step + 双层 stale-DOM 防护 + 失败计数器 + 预算门控
2. **软门控 vs 硬门控**：循环检测"只注入上下文 nudge、从不阻断"——LLM agent 的约束哲学
3. **上下文预算分级降级**：compaction（25 步/40k 字符）→ 75% 预算警告 → 最后一步强制 done → 失败强制 done
4. **事件驱动 watchdog 架构**：16 个监视器横切（安全/下载/captcha/权限/DOM/崩溃/HAR）——浏览器 agent 的"基础设施层"
5. **域级安全边界**：SecurityWatchdog 导航前/重定向/新标签三层域白名单 + 最小权限授予
6. **OSS 漏斗产品化第二实例**：与 mem0 同构（OSS 库→托管→自有模型）——跨项目模式候选

## 认知状态声明
- Fact/Observation：全部来自仓库实际内容（证据编号 E1-E25 可回溯）
- Pattern（L3）：本仓库验证 + 同类对照（对照项目：mem0/openclaw/agent-governance-toolkit，均为已考古 corpus 项目）
- Principle（L4+）：仅限本仓库证据的稳定关系；跨项目推广一律标 `Cross-project validation pending` 进 Candidates
