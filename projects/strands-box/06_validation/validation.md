# 06 Validation & Evidence — strands-box（ARCH-2026-10-23-001）

## Truth Audit（真值）
- 20 条 EK 全部回溯仓库实盘（HEAD 8c46dac，文件:行锚）；Cargo.toml 注释/AGENTS.md/decisions.md 为设计事实（已标注非实现事实）
- README/tenets 性能断言（<1ms）标注"asserted, not yet benchmarked"——未当已验证事实

## Coverage Audit（覆盖）
- 已覆盖：三拦截点（B01）/ 拒绝语义（B02）/ 引擎 pin（B03）/ 配置保护（B04）/ 类型化路径（B05）/ 统一事件（B06）/ MCP 双门（B07）/ temporal（B08）/ 回执（B09）/ 凭证网关（B10-B12）/ egress fold（B13）/ containment（B14）/ ShellSpawn（B15）/ 治理（B16/B18）/ 观察契约（B17）/ 测试体系（B19）/ Shell 成本（B20）
- 未深读（声明缺口）：dogwood-local-engine 引擎源码（外部 crate）、Linux namespace 全量实现（authority 已读部分）、mcp broker socket 身份、authority.rs ProposalState 细节、egress credential capability 全量、Monty 适配器（script adapter 仅见接口）

## Flow Audit（流真实性）
- 七类流全部从代码/文档导出；关键 Edge 可回溯：
  - 三适配器共享 Arc<PolicyEngine>（box/AGENTS.md "Three views of one authority"）
  - DenyReason 四分支（decision.rs:48-80）
  - reload 调用方认证（policy.rs:228-245 注释）
  - egress 机制失败即 Deny（decider.rs:14-28）
  - MCP gate2 非默认拒绝（decisions.md#mcp-authorization-is-two-gates）

## Abstraction Audit（升维诚实性）
- L4 4 个（KO-B01/B02/B03/B05）+ L3 1 个（KO-B04）——无 L5 硬凑
- KO-B05 为 cross-project（aegis 10-22 + Box 10-23 两仓库独立证据）——明确标注 Cross-project validation pending 的边界（文档降格 vs 自述标准两种形态已观察到，但"普遍定律"未宣称）
- 无单案例→Pattern 跳跃（C-B05/C-B07 保持 pending）

## Counterexample Hunt（反例）
- KO-B01 反例：MCP gate2 精化非默认拒绝（粗授权宽泛是设计）——默认拒绝有明确范围边界
- KO-B02 反例：无 per-process selection（同 box 内共享凭据路由——粗粒度取舍）+ operator 精确命名的 store 单点例外（B12）
- KO-B03 反例：egress Resolve 的 mcp_hosts PolicyPending 放行（白名单例外面）+ gate2 非拒绝——fail-closed 非绝对
- KO-B04/B05 反例：decisions.md 自警条目可落后（降格声明可证伪）

## Epistemic Audit（认知状态）
- C-B01~C-B07 全标 Hypothesis/Observation；无"结论漂亮就升层"
- 性能断言标注未基准（仓库自述 + 本考古未复测）
- 文档-实现对照：decisions.md 自称 historical record（与 aegis decisions 观对照）；AGENTS.md 冻结面有测试背书（name_is_required_and_every_removed_key_is_refused_by_name）

## 质量指标（初版）
- EK 20 条（B01~B20）/ KO 5 条（R1×1 R2×1 R3×2 R4×1）/ Candidates 7 / Flow 7 类 / evidence 引用 ~50 处
- EK links 覆盖率 100%；平均出边 ~1.7；游离 0%；聚合覆盖率 100%
- 反例预算：5 个 KO 各 ≥1 定向反例

## Contradictions 保留
- 无实现内部矛盾确认；已知**文档-实现缺口**：README "currently supports local execution on macOS" vs containment Linux namespace 后端存在（Linux 为 roadmap/部分实现——C-B01 追踪）；性能断言 vs box-bench exclude（未基准——C-B03 追踪）
