# 05 Candidates — browser-harness（未验证假说 / 暂定模式 / 跨项目假设）

> 与 KO 严格区分：以下内容不能确认、证据不足或跨项目未验证，绝不写成已验证 Principle。

## C-01 SKILL.md 身份冲突（仓库内部矛盾，需人工）
- 现象：SKILL.md front matter `name: browser-harness`；AGENTS.md 声明 "Skill identity (`name` + trigger) = `browser-use` (do not rename)"。
- 影响：安装/触发名在两个权威文档间不一致，可能影响 skill 分发与触发。
- 缺失证据：仓库内无统一身份声明；git history（浅克隆无历史）无法确认哪个先。
- 验证路径：查 browser-harness 安装脚本/发布流程；或向维护者确认。

## C-02 "多 agent 共享浏览器 lane"的并发安全边界（Hypothesis）
- 观察：SKILL.md 说多 agent 可串行共享 default daemon（"Sequential tab switching, input, and screenshot capture are safe"）；又警告两 agent 同时切 tab 会竞争。
- 假说：共享安全成立的前提是 agent 操作被外部序列化（无内置锁）。
- 缺失证据：仓库内无共享模式下的互斥实现（_dedicated_target_lock 只用于 named daemon 切换，不用于多 agent 排队）；无并发测试。
- 验证路径：双 agent 并发测试；或 SKILL.md 依赖的编排层（如 Codex）是否加锁。

## C-03 自动 cloud bootstrap 在真实 headless 环境的成功率（Hypothesis）
- 观察：test_run.py 固定 bootstrap 矩阵（BU_AUTOSPAWN 等 5 守卫），但真实无头服务器场景的端到端成功率未在仓库内验证（无集成测试）。
- 假说：无头环境（无 DISPLAY、无本地 Chrome）bootstrap 命中率高，但依赖 cloud API 可用性。
- 缺失证据：无 headless 集成测试；docs/ 仅 snap-linux-headless.md（本地 Chrome 主题）。

## C-04 "harness 自我改进"的长期漂移（跨项目假说）
- 观察：agent_helpers.py/domain-skills 随任务增长（AGENTS.md 策略），但无版本/兼容性治理（加载点注入所有公开名）。
- 假说：长期使用后 agent-workspace 会累积与核心 API 漂移的 helper（改名/废弃参数），_load_agent_helpers 的"所有非 _ 名注入 globals"策略可能遮蔽核心函数。
- 缺失证据：无长周期案例；仓库模板为空。
- 验证路径：多任务后检查 agent_helpers 与 helpers 的符号覆盖冲突。

## C-05 EK-45 Windows UTF-8 修复的触发场景（Observation→暂定）
- 观察：注释 #124(4) 表明特定 issue；reconfigure 在 run.py 启动路径。
- 不确定：是否所有 Windows locale（GBK/CP1252）都受影响；MCP server 路径是否同样需要（mcp_server.py 未见 reconfigure）。
- 验证路径：Windows 非 UTF-8 locale 下跑 browser-harness-mcp 复现。

## C-06 跨项目对照候选（暂定模式，不升 Principle）
- candidate-pattern-a：进程身份三重验证（P-02）与 deepseek-harness/opencode 的 daemon 身份机制是否同构——待跨 corpus 比对（S8 证据缺）。
- candidate-pattern-b：permission 拒绝=状态而非错误（CM-03）与 guardian/aigis 的权限模型交互——待比对。
- 状态：均标记 `Cross-project validation pending`，不写入 Generalized 层。

## C-07 recorder 自动录制偏好与隐私边界（需人工确认）
- 观察：auto_recording 是 opt-in 偏好（BH_RECORD 覆盖）；默认新装不录（SKILL.md）。
- 不确定：events.jsonl 的聚焦元素 box + input 标记（password 字段 mask）在真实登录墙场景是否充分——脱敏只对录制文本，不涉 DOM 原始数据（js() 结果可含敏感 DOM 文本，走 trace 时 500 字符截断但未正则清洗敏感实体）。
- 验证路径：录制含密码输入的真实流程，检查 events.jsonl 泄漏面。
