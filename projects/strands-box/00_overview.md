# 00 Overview — strands-box（ARCH-2026-10-23-001）

## 定位
AWS 官方开源（Apache-2.0）**OS 级 AI agent 行为遏制沙箱引擎**（"An open source sandbox engine for AI agents"）。
双层架构：**containment**（macOS Seatbelt / Linux namespace，进程遏制在 agent 进程外）+ **policy**（内嵌 Dogwood 本地引擎，决策默认拒绝）。
harness agnostic：`box run` 起任意 agent 程序/二进制，`box.toml` 声明访问，`policy.dw` 声明策略。

## 五条认知主线（本考古主张）
1. **fail-closed 是逐层机制而非顶层口号**：四拒绝语义（Forbidden/NoMatch/PolicyPending/InternalFault）+ cedar 安全 pin + SELF_DEFENDED_FILES + egress MutationCollision + ApprovedPath 类型化——每个可失败点都显式闭合
2. **执行面与决策面物理分离**：PEP（三适配器）与 PDP（唯一决策引擎）分离；enforcement 在 agent 进程外；agent 假设 hostile（tenet 1）
3. **机密性边界 = 边界持有**（tenet 2）：凭证 Vault 出站边界注入（5 种模式），真实密钥永不进 workload；FORBIDDEN_PATHS 永不授予；box 是凭证边界、无 per-process selection
4. **时间性策略 + 统一事件语义**：dogwood-local-engine durable history 支持 temporal 规则；fs:read/http:request 跨工具归一；拒绝回执 403 + rule id + description（agent 可自纠）
5. **治理即接口冻结**：interface-frozen（BLOCKER scrutiny）+ decisions.md（145 ADR anchor，"The code is the only authority"）+ 测试钉死行为（fail_closed_guards 防 fail-open 类回归）

## 关键文件锚（证据基座）
- README.md（Overview/Key capabilities）；AGENTS.md + crates/box/AGENTS.md（治理/冻结面）；docs/design/tenets.md（7 原则）；docs/design/decisions.md（145 ADR anchor）
- crates/policy/src/{policy.rs,request.rs,decision.rs,authority.rs,adapters/egress.rs,observe.rs}；crates/policy/Cargo.toml（引擎 pin）
- crates/credentials/src/{lib.rs,vault.rs,attach.rs,sources.rs}；crates/egress-gateway/src/capability/{decider.rs,credential/}
- crates/containment/src/backend/{macos/seatbelt.rs,linux/namespace/*}；crates/box/src/command/run.rs
- tests：crates/policy/tests/fail_closed_guards.rs；test-workload/（20 套件）；.github/workflows/（11 CI）
- 性能约束（tenets）：policy <1ms（"asserted, not yet benchmarked"——诚实标注未测）/ startup <50ms / mem <20MB

## 跨项目连接
- **dogwood（corpus）**：策略语言本体（Cedar 派生 + since/formerly/once）——Box 是 Dogwood 的**运行时载体**（dogwood-local-engine + dogwood-language =1.0.0 精确 pin）
- **MXC（10-19）**：平台级 OS 边界（AppContainer/WSLc/MicroVM 多后端 + 三模式）——Box = 开源单机侧（Seatbelt/namespace），同域两厂互补（对照详见 KnowlegeMap 卡 §5）
- **agent-aegis（10-22）**：进程内插桩治理（approval gate/策略引擎）——Box = 进程外 OS 遏制；两种治理位置
- **deepseek-harness（validated）**：sandbox/approval seam——Box 是 harness 无关的通用沙箱引擎
- **agent-governance-toolkit**：无状态策略引擎 vs Box 内嵌 PDP（fail-closed authority）
- **archon（10-21）**：控制面 gate vs Box 进程外 enforcement
