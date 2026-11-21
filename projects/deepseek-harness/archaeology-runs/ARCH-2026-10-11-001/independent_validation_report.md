# Independent Validation Report — deepseek-harness v0.2.1-alpha.2（盲重建）

> run_id: `ARCH-2026-10-11-001` ｜ Auditor：独立角色，**未把主考古 Package 当事实来源**，独立重读 `repo/` 建立 Independent Findings 后对比。
> 判定集：CONFIRMED / PARTIALLY_CONFIRMED / DOWNGRADED / OVER_GENERALIZED / MISSING / CONTRADICTED / NEEDS_HUMAN_REVIEW。**禁止修改原考古产物**（修正走阶段⑥ Reconciliation）。

## 一、盲重建方法

1. 独立重读仓库面（与主考古不同侧重）：`packages/core/agent-loop/src/agent.ts`（phase 状态机）、`scripts/verify-application-entrypoints.ts`（全文）、`packages/core/tools/src/index.ts`（工具管道事件）、`packages/schedule/schedule/src/domain.ts`（cron canonical）、`packages/extensions/cordis-host-runner/src/index.ts`（动态定义）、`packages/core/session/src/types.ts`（SESSION_FORMAT_VERSION 设计注释 + SessionHeader）、`packages/goal/goal/src/fold.ts`（phase 转换）、`packages/sandbox/sandbox-policy/src/`（网络策略字面）。
2. 建立 Independent Findings 后与主考古 EK/KO/Flow/Candidates 对比。
3. 重点攻击面：单案例→Pattern、Pattern→L4、项目经验→通用 Principle、ADR→实现事实、Flow Edge 真实性、bypass/override/exception/alternate/direct call/admin path/fallback/legacy path、Epistemic 状态混淆。

## 二、判定统计

| 判定 | 数量 | 条目 |
|---|---|---|
| CONFIRMED | 10 | I-01~I-10 |
| PARTIALLY_CONFIRMED | 2 | I-11（guard 默认启用）/ I-12（workspace-changes 开销）|
| DOWNGRADED | 1 | I-13（C-02 从"待验证"降为"更强缺证据"）|
| OVER_GENERALIZED | 1 | I-14（KO-06 scope 边缘）|
| MISSING | 3 | I-15（tool pipeline bypass）、I-16（兼容性审查笔记 10-08）、I-17（webworker-packer 例外）|
| CONTRADICTED | 0 | — |
| NEEDS_HUMAN_REVIEW | 1 | I-18（schedule cron 方言 vs 标准 cron 差分）|
| **合计** | **18** | — |

## 三、3 个成功（Independence 强确认）

### I-01 CONFIRMED — 启动纪律有可执行清单（非文档口号）
独立读 `scripts/verify-application-entrypoints.ts` 全文：`MANIFEST_BIN_ALLOWLIST` 仅 `apps/cli`（dsh=lib/bin.js）+ `packages/experimental/webworker-packer`（build-only）；`EXECUTABLE_SOURCE_ALLOWLIST` 为每个可执行源赋显式角色（`apps/cli/src/bin.ts`=supported launcher；subagent-{acp,claude-code,codex,dsh-sdk} fixtures=test-only；`python/sdk-runtime/runtime-bootstrap.mjs`=private packaging-only runtime dispatcher）。主考古 EK-18 成立且证据更强。
- 附带发现：subagent 四 provider（acp/claude-code/codex/dsh-sdk）以 test fixtures 形态被清单化——独立佐证 EK-30。

### I-02 CONFIRMED — agent-loop 是真实 phase 状态机（非仅文档）
独立读 `packages/core/agent-loop/src/agent.ts`：`private phase: Phase`（idle/running/maintenance）；`commitPhase` 发布外部可见状态转换；`abort.signal` + `wakeRequested`（wakeAfterAbort 在 disposed 之外唤醒）；"kick owns a running phase until this driver boundary"。主考古 EK-06/07/08 的 turn flow 有代码底座。Flow Atlas Control Edge（pre-step→prepareCall→提交）**可回溯到 agent.ts phase/abort/wake 结构**。

### I-03 CONFIRMED — goal phase 转换有显式校验（CAS 非口号）
独立读 `fold.ts`：`pause` 要求 `active→paused`、`block` 要求 `active→blocked`、`resume` 要求 resumable 且 `roundsStarted < maxGoalRounds`、`edit` 拒改 objective/maxGoalRounds。主考古 EK-20 的 CAS/phase 声明与实现逐条对应。

## 四、3 个错误/攻击（主考古问题）

### I-13 DOWNGRADED — C-02 的证据强度应下调（重要 negative 升级）
独立对 `packages/sandbox/sandbox-policy/src/` 做 network/fail-closed/failClosed 字面搜索：**0 命中**。主考古把 C-02 定为"缺失证据待验证"；盲重建把证据升级为**主动 negative**——sandbox-policy 源码层未出现网络策略关键词，即"默认 fail-closed 网络"在策略实现层无显式载体。Reconciliation 应把 C-02 表述改为"策略层主动 negative + 部署层要求"。

### I-15 MISSING — tool pipeline 的 bypass 路径（主考古遗漏）
独立读 `packages/core/tools/src/index.ts`：`tools/pre-execute`/`execute`/`post-execute` 三 waterfall 之后，源码明示 **"Policy replacements remain authoritative; pipeline failures that bypass post-execute skip projection"** ——存在显式 bypass 路径：管道失败绕过 post-execute 时跳过投影。主考古 EK-13 只写"把关管道（policy pre/post + guard）"，**未记录这条 bypass/exception 路径**。这是 Auditor 攻击面（bypass/fallback）上的实际遗漏。Reconciliation：EK-13 补充 bypass 语义；Flow Atlas Data Flow 补 edge。

### I-18 NEEDS_HUMAN_REVIEW — schedule cron 方言差分需人工确认
独立读 `domain.ts`：canonical 化实现（canonical RFC3339 UTC instant、canonical IANA zone）与 README 方言规则一致；但"dom/dow 双 star 才 AND、否则 OR"与主流 cron 语义（多数实现 AND）的差异影响未量化。主考古 C-07 标注 Observation——盲重建认为该差异**可能影响跨系统移植**，建议人工 review 是否升级为 Known Limitation。

## 五、遗漏（MISSING 汇总）

| ID | 遗漏 | 影响 | Reconciliation 动作 |
|---|---|---|---|
| I-15 | tool pipeline bypass post-execute（skip projection）| EK-13 不完整；Data Flow 缺 bypass edge | EK-13 补 bypass 语义；Flow Atlas 补 edge |
| I-16 | `.agents/notes/implemented/process/2026-10-08-session-reader-compatibility-review.md` 兼容性审查记录 | 主考古未引用该决策笔记（types.ts 注释明确指向它）| EK-02/03 补该证据引用 |
| I-17 | `webworker-packer` 是唯一非 cli 的 bin allowlist 成员（build-only）| 主考古 EK-18 未列该例外 | EK-18 补例外清单 |

## 六、Benchmark case（盲重建基准案例）

**基准案例：以"会话格式版本化治理"为 Gold Record 盲重建验证。**

- 盲重建独立从 `types.ts` 注释提取版本化设计规则：单调整数无 major/minor；bump 条件=现有 discriminator 无法阻止不安全解释（structural difference alone 不够）；未知字段/值需 released reader 实际 admission 行为的证据；`ignorable` marker 治理未知事件类型而非任意嵌套 payload；stored==current 时 adjacent migration chain 不运行；迁移需覆盖源的所有 supported representations。
- 与主考古 EK-02/03/KO-08 对比：**盲重建提取的规则集严格包含主考古的规则集**，且补充 3 条主考古未明示的规则（bump 条件、ignorable 语义、chain 不运行条件）→ 主考古 KO-08 无 OVER_GENERALIZED，但 EK-02/03 可补盲重建细节 → **CONFIRMED（部分细节补强）**。
- 判定：基准案例通过（Gold Record 可追溯性成立，blind 提取与主考古结论一致且更细）。

## 七、Epistemic 攻击结果

- 主考古 C-01/C-02 均标 Hypothesis：盲重建同意（仓库外/部署层证据，无冒充）✅
- 主考古 KO-06（单例 provider）标 L3：盲重建确认不升 L4（证据面仅两包）——**I-14 OVER_GENERALIZED 边缘**：KO-06 声明"设备/后端类能力"用单例约束，但仓库中 llm/subagent 是多 provider 的（contrast 已写）→ scope 已正确收窄，不成立 OVER_GENERALIZED，降级为 PARTIALLY_CONFIRMED（scope 正确但表述"设备/后端类"略宽）。
- 主考古 KO-10（可见性分层）标 Validated Pattern：盲重建同意（多包 README + tools bypass 中的 projection skip 语义亦佐证"模型不可见层"概念）✅

## 八、结论

- 盲重建独立 Findings 与主考古无 CONTRADICTED；1 处 DOWNGRADED（C-02 证据强度）、3 处 MISSING（I-15/16/17）、1 处 NEEDS_HUMAN_REVIEW（I-18）、1 处 PARTIALLY（I-14）。
- **主考古核心结论（KO-01~10）全部经受盲重建攻击**；修正项均非推翻性，属补强/收窄。
- 修正清单移交阶段⑥ Reconciliation（只改 Corpus Artifact，不改生产 Skill）。
