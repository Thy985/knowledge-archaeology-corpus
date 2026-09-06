# 05 · Candidates — Omnigent（未验证假说与跨项目候选）

> 不能确认的内容留在 Candidates。每条标注：假设状态 / 当前证据 / 缺失证据 / 验证路径。**Hypothesis 不得写成 Fact；Cross-project Candidate 不得写成已验证 Principle。**

## C-01（Hypothesis · 跨项目候选）"双评估"模式可推广到任意"执行近端 + 权威远端"系统
- **内容**：runner 本地 fast-path（ALLOW/DENY）+ server 权威（elicitation）的"双评估"可能是通用模式：凡执行在边缘、权威在中心的系统（边缘设备 agent、浏览器扩展、远端 runner），都可拆本地 fast-path 与权威通道。
- **当前证据**：Omnigent runner/policy.py（单项目，S3）。
- **缺失证据**：第二个项目实例。
- **验证路径**：对照 rampart（沙箱执行近端）或浏览器 harness 的权限模型；若出现相同双评估结构 → 升 L3。

## C-02（Hypothesis）"策略执行点跟随控制权"会在其他架构迁移中出现
- **内容**：任何把某控制权从 A 迁到 B 的重构（如 dispatch 从 server 移 runner），若策略不随迁则 parity 断裂——Omnigent 是实例，是否普遍未知。
- **当前证据**：EK-03 + RUNNER_MCP 决策史（单项目）。
- **缺失证据**：其他项目的迁移史对照。
- **验证路径**：检索同类 harness 项目（OpenClaw/OpenCode 等）的 dispatch 归属迁移 commit。

## C-03（Hypothesis）ASK 超时 86400s 的"默认信任人类"可能只适合交互式单用户场景
- **内容**：默认等待一天隐含"总会有人回来"；对无人值守编排（夜间批处理、agent 间调用）可能是隐患——虽然 headless 可显式 fail-closed，但默认值仍是 fail-open。
- **当前证据**：pending_approvals.py（S3）+ CHANGELOG 修复史（旧 120s 静默拒绝被认定为 bug）。
- **缺失证据**：无人值守场景的真实事故统计。
- **验证路径**：监控无人值守任务的 ASK 悬空时长分布。

## C-04（Hypothesis · 跨项目候选）"资源可替换、身份持久"适用于所有云代执行抽象
- **内容**：sandbox generation 可换而 host_id 绑定存活——猜想适用于容器、VM、serverless 的所有"身份 vs 资源"抽象（容器重启保留 IP？pod 换 node 保留 Service？）。
- **当前证据**：managed_hosts.py（S3）+ 工程直觉（K8s Service 类比）。
- **缺失证据**：与容器编排项目的系统对照。
- **验证路径**：对照 k8s/OpenShell 的绑定语义。

## C-05（Observation）OBSERVABILITY.md 的诚实审计可能被未来实现推翻
- **内容**：文档列出的所有缺口（trace 不传播、HTTPX instrumentor 未 wire、get_traceparent_env dead code）是快照时刻的事实；tracing 改造计划（proposed）若落地，这些"当前事实"会过时。
- **当前证据**：designs/OBSERVABILITY.md（S2，Status: Proposed）。
- **缺失证据**：后续 commit 中 tracing 是否已落地。
- **验证路径**：后续 refresh run 时 grep `get_traceparent_env` 调用点与 instrumentor 注册。

## C-06（Tentative Pattern）保守收割方向（拿不准就不删）可能是孤儿清理的通用安全准则
- **内容**：bridge 目录收割"保守在危险方向"（复用/外来 pid 视为活），且接受 check-then-rmtree 竞态（因活会话每 turn 刷新标记）。猜想：所有"清理不可靠环境的孤儿资源"都应默认保守。
- **当前证据**：native_bridge_common.py（S3）+ evolver stash 回滚的保守性（corpus 类比）。
- **缺失证据**：第三个项目实例。
- **验证路径**：对照其它 agent 系统的孤儿进程/临时目录清理策略。

## C-07（Unresolved Contradiction）tool_dispatch 的"原样上送"与 runner 本地策略执行的关系
- **内容**：tool_dispatch 说 action_required 事件"原样上送"保持可见性（executor 不自行分发），而 runner/policy.py 又说 runner 本地执行 function 策略——两者交界：policy gate 在分发前还是上送前？DENY 文本如何进入"原样上送"的事件流？
- **当前证据**：两文件 docstring 均提到对方但未在同一处说明顺序。
- **缺失证据**：proxy_stream 中 gate 与上送的确切调用顺序。
- **验证路径**：精读 runner/app.py 的 proxy_stream + tool_dispatch.execute_tool 调用点；需要时运行 tests/runner 相关测试。

## C-08（Scope-uncertain）server 与 runner 的"策略双面"是否会导致策略结果不一致
- **内容**：同一 policy 在 runner 与 server 各评估一次（双评估），若两侧 spec_hash 不一致或状态不同步，可能产生不同裁决；设计文档未明确两侧一致性保障（除 per spec_hash 缓存）。
- **当前证据**：runner/policy.py（_GatedPolicy per spec_hash 缓存）+ engine.py（server 侧）。
- **缺失证据**：两侧 spec_hash 同步机制与不一致时的裁决语义。
- **验证路径**：构造 runner/server spec 不一致的测试（对应 tests/policies/test_native_policy_hook.py 意图）。
