# Engineering Knowledge — deepseek-harness（EK Graph）

> Job: `ARCH-2026-09-03-001` | 层：Engineering Knowledge（宽底座，L1/L2）| **32 条 EK（Reconciled）**
> 26 条原始 + 6 条 Reconciliation 新增（EK-27..EK-32，来源：独立验证报告 M-1..M-6）。
> 每条 EK = 图节点，声明 `links`（六类边）；证据引用见 `06-validation/validation-summary.md`（EV-xxx）与独立验证 IF-xxx。
> 铁律：孤立 EK 要么被连接要么降级；本图 32/32 全部连接 KO 簇（游离 0%）。
> **Reconciliation 标注**：EK-07 已按独立验证修正管道顺序；新增 EK-27..32 标记 [R]。原 26 条保留不删（Engineering Knowledge 不因形成 KO 而删除）。

## 核心机制（Core Mechanisms）

### EK-01 — turn/step 双层驱动模型
- **claim**：agent-loop 用 turn（用户意图回合）+ step（一次模型请求及其工具调用）双层驱动；`turn/start→step/start→user/message→…→step/end→turn/end`，`agent/turn-stopping` 在回合末派发。
- **evidence**：EV-001, EV-002 | **abstraction**: L2 | **value**: A
- **epistemic_status**: Fact（代码实证）| **category**: RUNTIME | **confidence**: high
- **detail**：`ReactLoopAgent.runLoop` 循环内先 `preStep(target)`（inbox 认领），reject→`turn/end{blocked}`；空消息首步仍拥有 turn 边界但不消耗模型调用；`finally` 无条件 append `step/end`/`turn/end`，保证日志闭合。
- **flows**: control/state/data/memory | **scope**: Agent Runtime 会话循环
- **links**:
  - → EK-02（causal: 每次 append 写入 session log 事实源）
  - → EK-20（constraint: max-tokens sticky 约束回合结果）
  - ↔ EK-17（mechanism: 所有失败都在 finally 中结构化收尾）

### EK-02 — append-only SessionEvent 日志 = 唯一事实源
- **claim**：会话一切事实写入不可变 `SessionEvent` 日志（merge-extensible map），**"model-visible means logged"**；模型历史由 `deriveMessages()` 从日志派生，运行时禁止旁路状态。
- **evidence**：EV-003, EV-004 | **abstraction**: L2 | **value**: A | **confidence**: high
- **epistemic_status**: Principle（项目内运行时验证）| **category**: MEMORY
- **detail**：`SessionEventMap` 事件 lossless JSON + 连续 seq；`deriveMessages()` 按事件投影出模型可见历史；`deepFreeze` 保证不可变。
- **flows**: data/evidence/memory/policy
- **links**:
  - ← EK-01（causal in）→ EK-04（constraint: 非 surface 事件永不出现在模型 transcript）
  - → EK-05（causal: 投影从日志增量折叠）→ EK-06（dependency: 格式版本由日志格式约束）

### EK-03 — JSONL 只追加持久化 + 路径安全
- **claim**：session 日志以 JSONL（可选 zstd 压缩）只追加落盘；session id 经 `encodeSegment` 做路径段编码防 traversal；provenance 在存储边界 range-encode。
- **evidence**：EV-005 | **abstraction**: L1 | **value**: B | **confidence**: high
- **category**: PERSISTENCE/STORAGE | **epistemic_status**: Fact
- **flows**: data/memory
- **links**:
  - ← EK-02（dependency: 持久化的是同一事件流）→ EK-06（mechanism: 版本一致性）

### EK-04 — Session Surface：模型可见面是显式投影
- **claim**：模型可见面（surface）由事件带 `surfaceOp`（append/replace）显式标记；**替换范围被阴影遮蔽**；审计类事件（如 approval 决策）永不在模型 transcript，但模型可学到"策略"（approval/policy 事件重放）。
- **evidence**：EV-004, EV-006 | **abstraction**: L2 | **value**: A | **confidence**: high
- **category**: MEMORY/CONTEXT | **epistemic_status**: Fact
- **detail**：`surface.ts` `isSurfaceEvent`/`isAppendSurfaceEvent`；`approval/asked`+`approval/decided` 成对审计，永不进模型。
- **flows**: memory/policy/evidence
- **links**:
  - ← EK-02（constraint in: model-visible means logged）
  - → EK-19（causal: compaction 阴影替换 surface 范围）→ EK-11（subsystem: approval 审计事件同属 session 日志）

### EK-05 — Session Projection：派生状态增量折叠
- **claim**：投影（projection）不是存状态，而是注册单元对**已提交事件**做增量折叠，`stateOf()/snapshot()` 取派生值；agent-loop 注册 `turnBoundary` 共享投影；错误状态（disposed projection）独立追踪。
- **evidence**：EV-007, EV-008 | **abstraction**: L2 | **value**: A | **confidence**: high
- **category**: MEMORY | **epistemic_status**: Fact
- **flows**: state/memory
- **links**:
  - ← EK-02（causal in: 投影消费日志）→ EK-19（mechanism: 压缩也以投影/事件方式实现）

### EK-06 — 双版本纪律：格式 pinned vs schema 单调
- **claim**：`SESSION_FORMAT_VERSION = 0` 被 pin（无迁移承诺，未来破坏性演进）；SQLite `STORAGE_SQLITE_SCHEMA_VERSION = 1` 单调递增且启动时**不兼容即拒绝**（`PRAGMA user_version` 校验）。
- **evidence**：EV-009, EV-010 | **abstraction**: L2 | **value**: A | **confidence**: high
- **category**: ENGINEERING/DESIGN | **epistemic_status**: Fact
- **flows**: data/state
- **links**:
  - ↔ EK-03（mechanism: 同一事件流的存储层版本策略）
  - → EK-26（constraint: 版本破坏性演进需要 dsh 启动边界）

## 工具与执行（Tooling / Execution）

### EK-07 — 受控工具执行管线 [R：按独立验证修正]
- **claim**：工具执行真实顺序为 `collapse-check（确定性失败，先于策略管线）→ pre-execute waterfall → (gate.kind==='ask' → approval) → guards（monotonic denial，仅 decision.allow 时评估）→ execute → post-execute/final-result`；守卫只有拒绝没有允许；approval 在 pre-execute 与 guard 之间（由 pre-execute 返回 `ask` 触发）。
- **evidence**：EV-011, EV-012, IF-01（`packages/core/tools/src/index.ts:1367,1400,1468-1493`）| **abstraction**: L2 | **value**: A | **confidence**: high
- **category**: PERMISSION/TOOLING | **epistemic_status**: Fact
- **flows**: authority/control/data
- **links**:
  - → EK-09（causal: guards 在 approval 后评估，决定是否进入 execute）
  - → EK-10（dependency: 需要审批的动作依赖 approval 决策，approval 先于 guards）
  - → EK-08（mechanism: 同为执行阶段包装器）
  - → EK-28（causal: collapse-check 先于本管线，是 EK-28 的消费点）

### EK-08 — 协作式工具超时强制器
- **claim**：工具声明 `timeoutMs` 并承诺 honor `exec.signal`；`timeout-policy` 包装 arm 该 deadline，过期映射为结构化 `TOOL_TIMEOUT` 错误码，**不竞速不放弃工具 promise**；嵌套外层 deadline 不被误读（scope code 区分）。
- **evidence**：EV-013 | **abstraction**: L2 | **value**: B | **confidence**: high
- **category**: TOOLING | **epistemic_status**: Fact
- **flows**: control/authority
- **links**:
  - ↔ EK-07（mechanism: 同为 tools/execute 包装器）→ EK-17（dependency: TOOL_TIMEOUT 是结构化错误族一员）

### EK-09 — 守卫单调性：只有拒绝，没有允许
- **claim**：守卫的 `guardReason(exec)` 返回**第一个单调拒绝**，不允许跨层"允许"覆盖；无允许语义意味着默认不安全。
- **evidence**：EV-012 | **abstraction**: L2 | **value**: A | **confidence**: high
- **category**: PERMISSION | **epistemic_status**: Fact
- **flows**: authority
- **links**:
  - ← EK-07（causal in）→ EK-10（mechanism: 同为 fail-closed 家族）→ KO-03（R3 簇成员）

## 权限与安全（Permission / Security）

### EK-10 — Approval fail-closed
- **claim**：审批 Answerer 链失败/缺席 → 闭合 `'unavailable'`（fail-closed）；**非 `allowed-once` 即拒绝**；`approval/asked`+`approval/decided` 成对审计；授权只对请求的动作生效（grant is request-scoped）。
- **evidence**：EV-014, EV-015 | **abstraction**: L2 | **value**: A | **confidence**: high
- **category**: PERMISSION | **epistemic_status**: Principle（项目内验证）
- **flows**: authority/policy/evidence
- **links**:
  - → EK-11（causal: policy 决定 request 是否发出）→ EK-12（dependency: 审批通过后由沙箱执行边界落地）
  - ↔ EK-09（mechanism: fail-closed 家族）

### EK-11 — per-session ApprovalPolicy 持久化可重放
- **claim**：`ask/never` 策略 per-session 以 `approval/policy` 事件写入 session log，**重放可重建**；`setApprovalPolicy` 单一写路径；`never` 在任何 dispatch 前确定性拒绝（即使有恶意 listener）。
- **evidence**：EV-015, EV-016 | **abstraction**: L2 | **value**: A | **confidence**: high
- **category**: POLICY/PERMISSION | **epistemic_status**: Fact
- **flows**: policy/memory
- **links**:
  - ← EK-10（causal in）→ EK-14（constraint: presets 是策略的组合物）→ KO-06（R2 簇成员）

### EK-12 — SandboxMode 三态 + per-call 携带
- **claim**：`read-only / workspace-write / danger-full-access` 三态；**策略 per-call 携带而非 provider 固定**（同一 provider 同一时刻不同 consumer 可不同 mode）；enforcement `full/partial` 作为**上报事实**而非隐藏；网络不可达即可靠失败。
- **evidence**：EV-017, EV-018 | **abstraction**: L2 | **value**: A | **confidence**: high
- **category**: PERMISSION/SANDBOX | **epistemic_status**: Fact
- **flows**: authority
- **links**:
  - ← EK-10（dependency in: 沙箱是审批的执行边界）
  - → EK-13（mechanism: 都是 capability seam 的 provider 实例）→ EK-15（constraint: landlock 部分 ABI 影响上报诚实性）

### EK-13 — CredentialRef：引用而非值
- **claim**：settings/组合文件只携带**环境变量名引用**（CredentialRef branded），provider 拥有值；**每操作重解析**（轮换热生效）；空值处处视为不存在。
- **evidence**：EV-019, EV-020 | **abstraction**: L2 | **value**: A | **confidence**: high
- **category**: PERMISSION/CREDENTIAL | **epistemic_status**: Fact
- **flows**: authority/data
- **links**:
  - ↔ EK-12（mechanism: 同一 capability seam 三件套）→ KO-02（R1 簇成员）

### EK-14 — Permission Presets：两旋钮打包，自身不强制
- **claim**：预设把 sandbox+approval 两个旋钮打包为命名组合（`workspace-write`+`ask`、`danger-full-access`+`never`）；presets 自身**不执行强制**，只做声明与提供。
- **evidence**：EV-021 | **abstraction**: L1 | **value**: B | **confidence**: medium
- **category**: PERMISSION | **epistemic_status**: Observation
- **flows**: policy
- **links**:
  - → EK-11（constraint: 预设组合约束 approval policy）→ EK-12（constraint: 预设组合约束 sandbox mode）

### EK-15 — Landlock 部分 ABI 误分类教训（失败/修复）
- **claim**：broad 签名规则（`landlock-run: ` 前缀 + 任意非零退出）把子进程正常非零退出（rgrep 无匹配 exit 1）误分类为 `SANDBOX_UNAVAILABLE`；修复 = **status-gated fatal evidence** + 精确 informational exclusion + 关键路径无沙箱回归锁定。
- **evidence**：EV-022 | **abstraction**: L2 | **value**: A | **confidence**: high
- **category**: FAILURE | **epistemic_status**: Validated Pattern（postmortem 验证）
- **flows**: authority/evidence
- **links**:
  - → EK-17（mechanism: 结构化错误码的可信归属）→ EK-12（constraint in: partial enforcement 是 sandbox 上报事实）

## LLM 与适配（LLM / Adapters）

### EK-16 — LLM adapter 注册表 + 可插拔 provider
- **claim**：`ctx.llm` 是 adapter 注册表；provider 可插拔（deepseek / pi-ai / replay 录制回放）；统一消息/流词汇；adapterDefaults 可被请求级覆盖。
- **evidence**：EV-023, EV-024 | **abstraction**: L2 | **value**: A | **confidence**: high
- **category**: LLM | **epistemic_status**: Fact
- **flows**: control/data
- **links**:
  - ↔ EK-23（mechanism: 同为 capability seam 实例）→ KO-02（R1 簇成员）

### EK-17 — 结构化错误族
- **claim**：所有失败结构化——`LlmError` 保留 facts；非 LlmError 扁平化为 `errorChain` 文本 + `UNKNOWN` code；`TOOL_TIMEOUT`/`SANDBOX_UNAVAILABLE`/`SEARCH_FAILED` 等专用 code 供重试/回放路由。
- **evidence**：EV-025, EV-013 | **abstraction**: L2 | **value**: A | **confidence**: high
- **category**: FAILURE/RUNTIME | **epistemic_status**: Fact
- **flows**: control/evidence
- **links**:
  - ← EK-01（mechanism in）← EK-08（dependency in）← EK-15（mechanism in）→ KO-07（R1 簇）

## 上下文与记忆（Context / Memory）

### EK-18 — 有序 prompt section 组装
- **claim**：system prompt 由有序 section 注册（唯一名，重名抛错）+ 动态 context + 工具 schema + 变量组装；turn-boundary 组装；单次完整 section 独占（多则组装失败）。
- **evidence**：EV-026 | **abstraction**: L1 | **value**: B | **confidence**: high
- **category**: CONTEXT | **epistemic_status**: Fact
- **flows**: data/memory
- **links**:
  - → EK-04（causal: surface 提供模型可见消息源）→ KO-04（R4 簇成员）

### EK-19 — Compaction：替换范围为 summary 节点
- **claim**：压缩把历史范围替换为**一个 summary 节点**（shadow 语义）；`compaction/start` 持久事件；只有 idle agent 可压缩；history changed-span 竞态会被拒绝并记录。
- **evidence**：EV-027, EV-028 | **abstraction**: L2 | **value**: A | **confidence**: high
- **category**: MEMORY/COMPACTION | **epistemic_status**: Fact
- **flows**: memory/state
- **links**:
  - ← EK-04（causal in: 阴影替换 surface）← EK-05（mechanism in: 投影/事件方式）→ KO-04（R4 簇成员）

### EK-20 — max-tokens sticky
- **claim**：一旦某 step 达 max-tokens，后续正常完成的 step **不得把回合结果降级**（sticky 保持）；空首步/唤醒消息特殊处理。
- **evidence**：EV-029 | **abstraction**: L1 | **value**: B | **confidence**: high
- **category**: RUNTIME | **epistemic_status**: Fact
- **flows**: control/state
- **links**:
  - → EK-01（constraint in: 约束 turn 结果判定）→ KO-04（R4 簇成员）

## 子代理与编排（Subagent / Orchestration）

### EK-21 — Subagent 委托 seam（多 provider）
- **claim**：`tool-subagent` 通过 `ctx.subagents` 单一点委托，provider 可插拔（spawn / acp / claude-code / codex / dsh-sdk / fork-in-process）；model-facing tool 名与 provider 均可配置；`startContinuable` 后台续跑由 provider 决定。
- **evidence**：EV-030, EV-031 | **abstraction**: L2 | **value**: A | **confidence**: high
- **category**: ORCHESTRATION/AGENT | **epistemic_status**: Fact
- **flows**: control/authority
- **links**:
  - ↔ EK-23（mechanism: capability seam 实例）→ KO-02（R1 簇成员）

### EK-22 — 外部 Agent 集成（claude-code / codex hooks）
- **claim**：subagent 可委托给**外部 agent 进程**（@anthropic-ai/claude-agent-sdk、@openai/codex），通过 hooks 接入；证明"Agent Runtime 可异构编排"。
- **evidence**：EV-032 | **abstraction**: L2 | **value**: B | **confidence**: medium
- **category**: ORCHESTRATION | **epistemic_status**: Observation
- **flows**: control
- **links**:
  - → EK-21（subsystem: 同为 subagent provider 家族）→ KO-02（R1 簇成员）

## 架构与治理（Architecture / Governance）

### EK-23 — Capability Seam 三件套
- **claim**：每个可插拔能力 = Service Definition（Service/`ctx.xxx`）+ Provider（实现）+ Consumer（消费方）三件套；`docs/capability-seams.md` 由 `gen-doc-graphs.ts` **自动生成**依赖图（pkg→service→consumer）。
- **evidence**：EV-033, EV-034 | **abstraction**: L2 | **value**: A | **confidence**: high
- **category**: ARCHITECTURE | **epistemic_status**: Fact
- **flows**: control（装配）
- **links**:
  - → EK-16/EK-12/EK-13/EK-21（subsystem: 这些 seam 都是其实例）→ KO-02（R1 簇）
  - → EK-24（dependency: 依赖 Cordis context 树）

### EK-24 — Cordis 插件内核（everything-is-a-plugin）
- **claim**：运行时一切皆插件（vendor 化 Cordis）：context 树 + 服务注册 + scope；bundle = config rows + code；分层覆盖（bundle→profile→home→patch）。
- **evidence**：EV-035, EV-036 | **abstraction**: L2 | **value**: A | **confidence**: high
- **category**: ARCHITECTURE | **epistemic_status**: Fact
- **flows**: control
- **links**:
  - → EK-23（dependency in）→ EK-26（constraint: 启动边界由应用层强制）→ KO-08（R4 簇）

### EK-25 — Package-owned Invariants 注册表
- **claim**：`ctx.invariants` 可配置注册表：`package_allowlist/blocklist` 正则选择包；check 由各包自己注册（child installer fiber）；**启动时校验失败 loud（抛错）**；断言范围受 AGENTS.md conventions 约束（只能断言权威事件流/可变数据，不得断言服务存在）。
- **evidence**：EV-037 | **abstraction**: L2 | **value**: A | **confidence**: high
- **category**: GOVERNANCE/TESTING | **epistemic_status**: Fact
- **flows**: evidence/control
- **links**:
  - → EK-26（constraint: 启动强制是运行时不变量的一部分）→ KO-08（R4 簇成员）

### EK-26 — 应用启动强制（verify-application-entrypoints）
- **claim**：所有 bin/demo 显式分类（受支持应用/仅测试/废弃），`verify-application-entrypoints.ts` **拒绝绕过 `dsh` 的 Node 应用路径**；只有 dsh 命名 profile 启动受支持应用。
- **evidence**：EV-038 | **abstraction**: L1 | **value**: B | **confidence**: high
- **category**: GOVERNANCE/TOOLING | **epistemic_status**: Fact
- **flows**: control
- **links**:
  - ← EK-24（constraint in）← EK-25（constraint in）→ KO-08（R4 簇成员）

## Reconciliation 新增 EK（来源：独立验证报告 M-1..M-6）

### EK-27 [R] — approval 策略委派继承（delegation override seeding）
- **claim**：approval policy 的 effective 值是两层的——`effectivePolicy = overrideOf(session) ?? config.policy ?? 'ask'`；session override 以 `approval/policy` 事件持久化；**子会话委派时用 `source:'delegation'` 播种 override**，实现 subagent 策略继承（authority boundary 跨进程传递）。
- **evidence**：IF-03（`packages/interaction/user-approval/src/index.ts:29-35,87-89,236,244`）| **abstraction**: L2 | **value**: A | **confidence**: high
- **category**: POLICY/PERMISSION | **epistemic_status**: Fact
- **flows**: authority/policy
- **links**:
  - ↔ EK-11（subsystem: 同为 approval policy 机制；补充 config→override 两层解析）
  - → KO-06（R2 簇成员，强化闭环的跨会话传递）

### EK-28 [R] — PTC collapse 隐藏契约：确定性失败先于策略管线
- **claim**：collapsed tool call（run_code/PTC 内直接调用被折叠的工具）**在策略管线（pre-execute/approval/guards）之前确定性拒绝**——"策略监听器不得观察、更不得批准一个注定失败的调用"；拒绝携带路由提示 `ToolNotFoundError`（`only run_code is callable directly`）。
- **evidence**：IF-04（`packages/core/tools/src/index.ts:1367-1411`）| **abstraction**: L2 | **value**: A | **confidence**: high
- **category**: PERMISSION/TOOLING | **epistemic_status**: Fact
- **flows**: authority/control
- **links**:
  - ← EK-07（causal in: collapse-check 是 EK-07 管线的前置门）
  - ↔ EK-09（mechanism: 同为"默认拒绝"；collapse 是比 guard 更早的确定性拒绝）
  - → KO-03（R3 簇成员）

### EK-29 [R] — plan-mode per-agent 协作状态
- **claim**：plan mode 是 **per-agent 协作状态**：`plan` projection 折叠 session log（resume/fork 恢复）；`plan/mode` 事件从 `agent/pre-step` 追加（仅 step 被接受时）；**sandbox 与 approval 独立于 plan state**（不读写 plan 状态）；进入/退出只改 prompt section，不改工具目录。
- **evidence**：IF-05（`packages/plan/plan-mode/src/index.ts`）| **abstraction**: L2 | **value**: A | **confidence**: high
- **category**: STATE/COLLABORATION | **epistemic_status**: Fact
- **flows**: state/memory
- **links**:
  - ↔ EK-05（mechanism: 同为 projection 折叠 session log 的实例）
  - → KO-04（R4 簇成员）

### EK-30 [R] — settings seam 分层解析 + 机密脱敏
- **claim**：用户设置 seam（`ctx.settings`）按 `schema defaults → registrant base → user document` 三层解析；`redactSecrets` 对值做机密脱敏；namespace 命名受限（`/^[a-z][a-z0-9-]*$/`）。
- **evidence**：IF-06（`packages/settings/settings/src/index.ts`）| **abstraction**: L1 | **value**: B | **confidence**: high
- **category**: CONFIG/SETTINGS | **epistemic_status**: Fact
- **flows**: data
- **links**:
  - ↔ EK-13（mechanism: 同为配置/机密分离 seam；settings 存结构，credentials 存引用）
  - → KO-02（R1 簇成员）

### EK-31 [R] — repeat-tool-reminder guard（advisory 反馈）
- **claim**：guard 包第二成员 `repeat-tool-reminder` 是 **advisory 型重复调用检测器**：丰富 post-execute 决策、写入模型可见的日志上下文，**不否决不改写调用**；阈值配置 load-time loud 失败（非法配置抛错而非静默降级）。
- **evidence**：IF-07（`packages/guard/repeat-tool-reminder/src/index.ts`）| **abstraction**: L1 | **value**: B | **confidence**: high
- **category**: TOOLING/GOVERNANCE | **epistemic_status**: Fact
- **flows**: feedback/control
- **links**:
  - ↔ EK-08（subsystem: 同 guard 家族）
  - → KO-08（R4 簇成员，作为运行时 advisory 治理）

### EK-32 [R] — legacy 格式硬拒绝策略
- **claim**：session 对 `request/header-delta` 旧格式与 `reason:'fallback'` 旧原因**硬拒绝**（抛错而非静默支持）——兼容策略是"拒绝而非支持"，保证格式演进的显式边界；`TurnEndCancelCause` 保留 `legacy` 变体。
- **evidence**：IF-09（`packages/core/session/src/index.ts:216,362-370`）| **abstraction**: L1 | **value**: B | **confidence**: high
- **category**: PERSISTENCE/COMPAT | **epistemic_status**: Fact
- **flows**: data/failure
- **links**:
  - ↔ EK-06（mechanism: 同为版本/格式演进纪律）
  - → KO-01（R2 簇成员）

## 附录：Evidence → EK 索引（完整 EV 清单见 06）
- EV-001..002: agent-loop runLoop（turn/step 事件序列）— packages/core/agent-loop/src/agent.ts:256-400
- EV-003..004: SessionEventMap / model-visible means logged — packages/core/session/src/types.ts:261, index.ts:790
- EV-005: JSONL format / path encode — packages/session/session-persistence-jsonl/src/format.ts:37,154,234
- EV-006: surface.ts — packages/core/session/src/surface.ts:29-67
- EV-007..008: projection — docs/architecture.md §Session log / session-projection
- EV-009..010: SESSION_FORMAT_VERSION / SCHEMA_VERSION — session/types.ts:87, storage-sqlite/schema.ts:20,82
- EV-011..012: tools pipeline / guards — packages/core/tools/src/index.ts:144,167,697-741
- EV-013: timeout-policy — packages/guard/timeout-policy/src/index.ts
- EV-014..016: approval — packages/interaction/user-approval/src/index.ts:48,207,258-264
- EV-017..018: sandbox — packages/sandbox/sandbox/src/index.ts:29,62-71,103
- EV-019..020: credentials — packages/credentials/credentials/src/index.ts:2,29,117
- EV-021: permission-presets — docs/subsystems/permission-presets.md
- EV-022: postmortem 0004 — docs/postmortem/0004-landlock-partial-notice-misclassified-child-failures.md
- EV-023..024: llm registry — docs/capability-seams.md (ctx.llm) / packages/llm/llm
- EV-025: structured errors — agent-loop/src/agent.ts:394-400
- EV-026: system-prompt sections — packages/core/system-prompt/src/index.ts:2,52
- EV-027..028: compaction — packages/compaction/compaction/src/index.ts:120-162
- EV-029: max-tokens sticky — agent-loop/src/agent.ts:280-302
- EV-030..031: subagent — packages/subagent/tool-subagent/src/index.ts:43-67
- EV-032: hooks-claude-code/codex — packages/hooks/*
- EV-033..034: capability-seams graph — docs/capability-seams.md / scripts/gen-doc-graphs.ts
- EV-035..036: Cordis vendor — vendor/README.md / pnpm-workspace.yaml
- EV-037: invariants — packages/runtime-diagnostics/invariants/src/index.ts
- EV-038: verify-application-entrypoints — scripts/verify-application-entrypoints.ts / docs/architecture.md
