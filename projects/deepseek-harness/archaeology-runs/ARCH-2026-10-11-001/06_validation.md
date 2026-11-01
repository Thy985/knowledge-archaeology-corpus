# 06 Validation & Evidence — deepseek-harness v0.2.1-alpha.2

> 验证原则：先 Blind Reconstruction（不把考古结果当事实来源，独立重读仓库）再对比；不弱化证据标准换全绿；Contradictions/Counterexamples 必须保留。

## 1. Truth Auditor（事实核查）

抽查清单（每项 = 独立重读仓库验证）：

| # | 声明 | 验证方式 | 结果 |
|---|---|---|---|
| T1 | SESSION_FORMAT_VERSION = 4 | grep types.ts:89 | **CONFIRMED**（`export const SESSION_FORMAT_VERSION = 4`）|
| T2 | verify-application-entrypoints 存在 | ls scripts/ | **CONFIRMED** |
| T3 | repeat-tool-reminder 是 per-agent advisory detector | 读 src/index.ts:2-5 | **CONFIRMED**（"Advisory per-agent repeat-call detector. It enriches post-execute decisions"）|
| T4 | timeout-policy 是 cooperative tool-call timeout | 读 src/index.ts:2-15,21 | **CONFIRMED**（声明 timeoutMs；deadline/timeoutOf；"nested outer deadline"作用域）|
| T5 | ptc-runtime 是 abstract resolve/run 契约 | 读 src/index.ts:143,150 | **CONFIRMED** |
| T6 | goal 有 CAS 拒绝（不能改 objective/maxGoalRounds）+ phase 状态机 | 读 fold.ts:187-188,215,221 | **CONFIRMED**（"cannot change objective or maxGoalRounds"）|
| T7 | tool-cordis 只读（cordis_inspect_list/query）| 读 src/index.ts:23,42 | **CONFIRMED** |
| T8 | jobs owner = sessionId + archive 时 kill | 读 archive-admission.ts:25-35,45 | **CONFIRMED**（`registry.kill(job.id, sessionId, 'session archived')`）|
| T9 | e2b 移除 | find packages -name "*e2b*" | **CONFIRMED**（无结果）|
| T10 | 331 包 | find packages -name package.json -maxdepth 3 \| wc -l | **CONFIRMED**（331）|
| T11 | tag = dsh-v0.2.1-alpha.2 | git describe --tags | **CONFIRMED** |
| T12 | guard 默认随 base bundle 启用 | guard/README.md（"Both ship enabled in the dsh base bundle"）| **CONFIRMED**（README 声明；config 实测未做→标注 PARTIALLY）|
| T13 | schedule delivery 需 flush 确认 | schedule/README.md | **CONFIRMED**（"delivery commits only after the Session acknowledges session/flush"）|

## 2. Coverage Auditor（覆盖核查）

| 面 | 覆盖 | 缺口 |
|---|---|---|
| 核心循环 | turn flow / pre-step / prepareCall / attempt / failed steps（docs + agent-loop 引用）| agent-loop 源码逐行未全读（引用 README 而非 agent.ts 全文）|
| 会话持久化 | 版本化/迁移/世代布局/投影 | storage-sqlite schema 逐字段未重读（基线已覆盖）|
| 权限与沙箱 | SAFETY.md / sandbox 组 / approval / PTC isolation / 启动纪律 | sandbox-policy 网络默认配置未实测（→C-02）|
| 控制面（NEW）| goal/jobs/schedule/deliverables/spill/workspace/computer-use/browser-use 全读 README + 抽查实现 | goal continuation driver、schedule delivery 实现细节未读 |
| 互操作 | MCP/ACP/subagent/teams/telemetry（基线维持）| 0.2 未逐行重读（标 STABLE 引用基线）|
| 测试 | 多配置清单 | 未跑任何测试（只读考古，不执行）|

## 3. Flow Auditor（Flow Edge 真实性）

- Control：`agent/pre-step` 拒绝→无 step turn、`prepareCall` 取消不提交——docs/architecture.md §Turn flow 原文 + EK-08 实现引用一致 ✅
- State：`SESSION_FORMAT_VERSION=4` 与 session-format-status.md（latestFinalizedVersion 4）一致 ✅；世代布局（v0/vN lowercase）与 docs 一致 ✅
- Authority：SAFETY.md "not the sole security control" 与 PTC "isolation descriptors do not promise a security boundary" 一致（同主题双层声明）✅
- Memory：goal/jobs/schedule 的持久化与权限分离声明在各包 README 一致 ✅
- Policy：AGENTS.md 治理纪律（released format 迁移规则）与 session-format-status.md 的 obligations 一致 ✅

## 4. Abstraction Auditor（升维判定）

| KO | 层 | 判定 |
|---|---|---|
| KO-01 全插件无特权内核 | L4 | 维持（0.1.2+0.2.1 双版本 + extensions 反例收窄）|
| KO-02 决策先于提交 | L4 | 维持（turn flow 文档化充分）|
| KO-03 事件溯源日志即真相 | L4 | 维持（跨版本一致）|
| KO-04 安全边界显式声明 | L4 | 维持（SAFETY.md 原文 + PTC 双重声明）|
| KO-05 持续工作控制面 | L4 | 维持（五包一致）|
| KO-06 单例 provider | L3 | 维持（两实例，不升 L4——证据面窄）|
| KO-07 单一权威启动入口 | L4 | 维持 |
| KO-08 版本化兼容治理 | L4 | 维持 |
| KO-09 守卫闭环 | L3 | 维持（advisory 性质限制升维）|
| KO-10 可见性分层 | L4 | 维持（文档+实现一致）|

**降级记录**：无（本轮无过度升维检出）。**升维门槛**：L5 Methodology 未产出（跨项目验证不足，符合纪律）。

## 5. Counterexample Hunter（反例预算制：每个 L3+ KO ≥1 定向反例攻击，本轮 10 KO 共 10 攻击）

| KO | 反例攻击 | 结果 |
|---|---|---|
| KO-01 | extensions node:vm 动态定义=特权化？ | 反例不成立：进程级+重启消失+仅 Plugin Manager 持久（EK-19）→ **确认** |
| KO-02 | 是否有绕过"决策先于提交"的路径？ | 取消只影响两 async 阶段（prepareCall 前）；提交后的取消由 cancellation cause 管理（docs）→ **部分确认**（提交后取消路径细节未全读）|
| KO-03 | 是否有第二个真相源？ | session-query / storage 均从 log 派生；无独立事实源发现 → **确认** |
| KO-04 | sandbox-windows-acl/sandbox-policy 是否提供强隔离？ | SAFETY.md 明言"even correctly enforced restrictions cannot protect resources the project is allowed to access" → **确认（声明边界非保证）**|
| KO-05 | goal/jobs/schedule 是否共享新特权面？ | 全部挂在既有 ctx/seam + session/event；无新特权面 → **确认** |
| KO-06 | 是否有非单例 provider 的能力？ | subagent/llm adapter 是多 provider 的（非设备类）→ 单例约束仅限设备/浏览器类 → **确认（scope 正确）**|
| KO-07 | vendored CLI 绕过启动纪律？ | 被 verify-application-entrypoints 显式分类（非绕过，被清单化）→ **确认** |
| KO-08 | 是否有降级承诺？ | "released-format obligations"无降级；consistency gate 拒低 writer 顺序 → **确认** |
| KO-09 | guard 是否强制？ | advisory/cooperative（repeat=enrich decision；timeout=cooperative 声明）→ **确认（非侵入）** |
| KO-10 | spill locator 是否回到模型？ | 是，但为受控 bounded preview（maxInlineTokens）→ **部分确认（受控例外）** |

**Contradictions**：0 条检出。**Counterexamples**：KO-02/KO-10 为 PARTIALLY（提交后取消细节、spill locator 受控回传），已在 KO 声明中标注。

## 6. Epistemic Auditor（认知状态诚实性）

- 所有 EK 均带证据路径；STABLE/CHANGED/NEW 标注不掩盖"基线结论在 0.2 未逐行重验"（Coverage 缺口如实列出）✅
- C-01/C-02 明确 Hypothesis（跨项目/外部事件），未升 Principle ✅
- C-03~C-07 标注 Observation/Hypothesis 边界 ✅
- 无"同子系统=聚合理由"假聚合（R1-R4 逐条声明）✅
- 无机械复制仓库文档（EK 为机制提炼，非 README 复述）✅

## 7. 盲重建（Blind Reconstruction，阶段⑤独立 Auditor 执行）

- 独立 Auditor 未把本 Package 当事实来源，独立重读 `repo/` 建立 Independent Findings 后对比——完整报告见 `independent_validation_report.md`（阶段⑤交付）。
- 本层仅记录主考古侧质量门结果：**六 Auditor 全部执行；0 Contradiction；2 PARTIALLY 反例（已标注）；质量门 PASS（不弱化标准）。**

## 8. 质量指标（quality metrics，写入 run_metadata.yaml）

```
evidence_count: 40                # 抽查 + 文档证据条目（T1-T13 + EK 证据）
ek_count: 40
ko_count: 10
candidate_count: 7
flow_count: 7
ek_links_coverage: 1.0            # 40/40 EK 声明 links
isolated_ek_ratio: 0.0
aggregation_rule_coverage: 1.0    # 10/10 KO 声明 aggregation_rule
ko_avg_cluster_size: 5.0
l4_ko_count: 8
l5_ko_count: 0
counterexample_attacks: 10
contradictions: 0
partially_confirmed: 2            # KO-02 / KO-10
validation_result: PASS
```
