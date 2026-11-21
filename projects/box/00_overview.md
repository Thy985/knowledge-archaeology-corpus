# Strands Box — Project Archaeology Overview（ARCH-2026-10-20-001）

## 一句话定位
AWS 开源的 OS 级 AI agent 行为遏制沙箱（Apache-2.0，Rust）：以操作系统隔离（macOS Seatbelt 当前 / Linux namespace 代码在）建立硬边界，叠加 Dogwood 策略引擎（Cedar 超集 + 时间性规则）对 Strands Shell / Monty（Python）/ egress gateway / MCP broker 四类 enforcement point 的每个请求做语义放行/拒绝。**核心命题：沙箱限"能触达什么"，策略限"能对触达之物做什么"；凭据永远不进 agent 进程。**

## 认知主线（本考古回答"这个项目让我们认识到什么"）
1. **策略是唯一授权者，其余一切只是 floors**：`CapabilityOutcome{Applied,Unavailable}` 连 Allow/Deny 都没有，机制不可能授权；deny-only floors 只能减不能加
2. **凭据边界 = 注入点，不是授权**：agent 拿占位符，gateway 在网络边界换入真实密钥（env:// / aws:// 两 scheme），"credential binding is configuration, not a policy action"
3. **时间性策略 = 跨调用因果约束**：单引擎共享历史让"Shell 读了敏感文件 → 5 分钟内拒绝出站 HTTP"成为可表达规则；历史按 box 持久、无回退、非审计记录
4. **执行面与决策面分离**：解释器/网关在可信进程（沙箱外）；alias 只转发无权力；trampoline 自遏制后 exec 目标（macOS 沙箱单向不可撤销）

## 关键文件锚（evidence 引用基线）
- 设计锚全集：docs/design/decisions.md（1913 行，决策锚点 ~90 条）
- 架构：docs/design/architecture.md / containment.md / credentials.md / egress.md / policy.md / telemetry.md / mcp.md / monty.md / shell.md
- 信条：docs/design/tenets.md（7 tenets + 3 non-goals + 性能目标）
- 自我治理：AGENTS.md（interface-frozen + 9 premises）
- 源码：crates/{box,policy,containment,egress-gateway,credentials,shell,telemetry}/src
- 测试：test-integ/（黑盒确定性遏制）、test-workload/、test-native-filesystem/

## 跨项目连接
- **dogwood**（corpus 09-08）：Box 内嵌的策略语言本体（Cedar 超集+时间算子）——本考古为"语言+宿主"同生态第二半
- **MXC**（corpus 10-19）：同域对照（Microsoft JSON 策略+多后端谱系 vs AWS Cedar/Dogwood 语法+Seatbelt）——周五验证计划 MXC×Box 双对照
- **deepseek-harness**：sandbox/approval seam 应用层 vs Box OS 层
- **agent-governance-toolkit**：服务端无状态策略引擎 vs Box 本地 PDP
- 用户项目：TeamMind（多 Agent 信任边界）、Tafcm（本地执行遏制）、silver-shield（确定性出站管控）、campus_order（OpenClaw 遏制）
