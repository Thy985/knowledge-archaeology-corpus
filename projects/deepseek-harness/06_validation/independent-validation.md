# Independent Validation Report — deepseek-harness (ARCH-2026-09-03-001)

> 独立 Auditor 报告 | 协议：Blind Reconstruction（阶段 A 独立重读 → 阶段 B 对比）
> 未将原 Archaeology Result 作为事实来源；未修改原考古产物。
> Repository is the Source of Truth：与独立证据冲突时以仓库实际代码为准。

---

## 一、Independent Findings（阶段 A：独立重读仓库所得，未参考原考古）

- **IF-01 工具执行管道真实顺序**（`packages/core/tools/src/index.ts:1367-1493`）：
  `collapse-check（确定性失败，先于策略管线）→ tools/pre-execute waterfall → (gate.kind==='ask' → approval serviceAsk) → guards(仅 decision.allow 时 guardReason) → execute → post-execute/final-result`
- **IF-02 approval 的 'never' 不可绕过性有专门测试**（`interaction/user-approval/tests/approval.spec.ts:430`）："never is unbypassable even by an answerer PREPENDED after the service mounts"。
- **IF-03 approval policy 是两层的**：`effectivePolicy = overrideOf(session) ?? config.policy ?? 'ask'`（`user-approval/src/index.ts:236`）；`approval/policy` override 是 session 单一持久化覆盖；**override 有 `source: 'delegation'` 变体——子会话委派时播种策略**（`index.ts:29-35,87-89,244`）。
- **IF-04 collapsed tool call（PTC/run_code）**在策略管线之前确定性拒绝："策略监听器不得观察——更糟的是批准——一个注定失败的调用"（`tools/index.ts:1367`）；带路由提示的 `ToolNotFoundError`。
- **IF-05 plan-mode 是 per-agent 协作状态**：`plan` projection 折叠 session log（resume/fork 恢复）；`plan/mode` 从 `agent/pre-step` 追加（仅 step 被接受时）；**sandbox 与 approval 独立于 plan state**（`packages/plan/plan-mode/src/index.ts`）。
- **IF-06 settings seam 分层解析**：schema defaults → registrant base → user document；`redactSecrets` 机密脱敏（`packages/settings/settings/src/index.ts`）。
- **IF-07 guard 包实际含两个成员**：`timeout-policy` + `repeat-tool-reminder`（advisory，不否决不改写，丰富 post-execute 决策）。
- **IF-08 多个未覆盖子系统**：`goal`/`schedule`/`spill`/`mcp`/`feedback`/`jobs`/`code-runtime`/`client`/`host`/`workspace`/`session-query` 等。
- **IF-09 legacy 兼容策略**：session 对 `request/header-delta` 旧格式与 `reason:'fallback'` **硬拒绝**（`session/src/index.ts:216,362-370`）——拒绝而非支持。
- **IF-10 `agent.inject` 写日志**（`agent/src/runtime-types.ts:147-149` 为 UserMessage，进入 session log）→ **非旁路**，model-visible means logged 无直接反例。

---

## 二、对比判定（阶段 B：Independent Findings × Archaeology Result）

### 逐对象判定（26 EK / 8 KO / 7 Flow）

| 对象 | 判定 | 依据 |
|---|---|---|
| EK-01 turn/step 驱动 | **CONFIRMED** | runLoop 实证（agent.ts:256-370） |
| EK-02 session log 事实源 | **CONFIRMED** | types.ts:261 + surface.ts:51 |
| EK-03 JSONL 持久化 | **CONFIRMED** | format.ts |
| EK-04 session surface | **CONFIRMED** | surface.ts:29-74（shadow 语义实证） |
| EK-05 session projection | **CONFIRMED** | IF-05 另证实 plan projection 同类 |
| EK-06 双版本纪律 | **CONFIRMED** | SESSION_FORMAT_VERSION/SCHEMA_VERSION |
| **EK-07 工具管线** | **PARTIALLY_CONFIRMED** | IF-01：遗漏 approval 精确位置（pre-execute 与 guard 之间）+ 遗漏 collapse-check 先于策略管线 |
| **EK-08 guard** | **PARTIALLY_CONFIRMED** | IF-07：只覆盖 timeout-policy，遗漏 repeat-tool-reminder |
| EK-09 守卫单调性 | **CONFIRMED** | scoped.spec.ts:303 实证：pre-execute force allow 不能 bypass guard |
| EK-10 approval fail-closed | **CONFIRMED** | IF-02 测试实证 |
| **EK-11 per-session policy** | **PARTIALLY_CONFIRMED** | IF-03：遗漏 config.policy 默认层 + delegation 播种继承 |
| EK-12 sandbox 三态 | **CONFIRMED** | sandbox/src + win32-process 实证 |
| EK-13 CredentialRef | **CONFIRMED** | credentials/src |
| EK-14 presets 不强制 | **CONFIRMED** | 独立核查无强制代码（Observation 状态正确） |
| EK-15 landlock 教训 | **CONFIRMED** | postmortem 0004 |
| EK-16 LLM 注册表 | **CONFIRMED** | capability-seams（ctx.llm） |
| EK-17 结构化错误族 | **CONFIRMED** | agent.ts:394-400 + TOOL_TIMEOUT |
| EK-18 有序 section | **CONFIRMED** | system-prompt/src |
| EK-19 compaction | **CONFIRMED** | compaction/src |
| EK-20 max-tokens sticky | **CONFIRMED** | agent.ts:280-302 |
| EK-21 subagent seam | **CONFIRMED** | tool-subagent/src |
| EK-22 外部 agent 集成 | **CONFIRMED**（Observation） | hooks 包 |
| EK-23 Capability Seam | **CONFIRMED** | capability-seams.md（generated） |
| EK-24 Cordis 插件内核 | **CONFIRMED** | vendor/README |
| EK-25 invariants | **CONFIRMED** | invariants/src |
| EK-26 启动强制 | **CONFIRMED** | verify-application-entrypoints |
| **KO-01 日志驱动循环** | **CONFIRMED** | IF-10 补充：inject 也走日志 |
| KO-02 Capability Seam 簇 | **CONFIRMED** | 多 seam 实例 |
| **KO-03 fail-closed 族** | **PARTIALLY_CONFIRMED（建议 scope 收紧）** | IF-01：approval 只覆盖 gate.kind==='ask'；sandbox 只覆盖 confined 路径；"任意一层失败都不产生授权"需限定为"受限执行路径" |
| **KO-04 四层记忆** | **PARTIALLY_CONFIRMED** | IF-05：遗漏 plan projection 作为第五类"协作状态投影"实例；四层模型本身正确 |
| **KO-05 日志即真相** | **PARTIALLY_CONFIRMED** | 主主张成立（IF-10），但"无旁路状态"隐含主张过宽：compaction 有损、settings 等配置不在日志中 |
| KO-06 策略治理闭环 | **CONFIRMED** | IF-03 补充 delegation 播种，强化闭环 |
| KO-07 结构化错误族 | **CONFIRMED** | 已条件化（SEARCH_FAILED） |
| KO-08 可插拔治理护栏 | **CONFIRMED** | 四护栏实证 |
| F-01..F-04, F-06, F-07 | **CONFIRMED** | symbol 逐条核实 |
| **F-05 Authority** | **INCORRECT（部分）** | IF-01：approval 在 guard **之前**（非之后）；且遗漏 collapse 先于策略管线 |

### 判定统计
- CONFIRMED：**34**（23 EK + 5 KO + 6 Flow）
- PARTIALLY_CONFIRMED：**6**（EK-07, EK-08, EK-11, KO-03, KO-04, KO-05）
- INCORRECT：**1**（F-05 Authority 管道顺序）
- DOWNGRADED：**1**（KO-03 → 建议将 L4 claim 收紧为"受限执行路径上的分层 fail-closed"，仍 L4 但 scope 明确；与 PARTIALLY_CONFIRMED 同一对象，作附加建议不重复计数）
- OVER_GENERALIZED：**0**
- MISSING：**7 项工程知识**（见下）
- CONTRADICTED：**0**
- NEEDS_HUMAN_REVIEW：**0**（管道顺序已代码实证，无需人工仲裁）

---

## 三、特别回答

### 3.1 原 Archaeology 最重要的 3 个成功
1. **"日志即真相"（model-visible means logged）的提炼**——是项目最独特、最可迁移的核心洞见，代码（surface.ts shadow 语义 + deriveMessages + agent.inject 也走日志）与测试双重实证，且组织为 KO-05 的 L4 Principle（已诚实标 Cross-project pending）。
2. **Capability Seam 三件套作为架构主轴**——从 `capability-seams.md`（自动生成图）识别出 llm/sandbox/credentials/subagent 多个独立实例，聚合为 KO-02 的 R1 机制簇，抓准了项目"可插拔是主轴而非装饰"的本质。
3. **fail-closed 权限族的正确分层**——审批/守卫/沙箱/凭证四层提炼准确，且"守卫只有拒绝没有允许"（EK-09）被独立测试证实（pre-execute force allow 不能 bypass guard），"never unbypassable"（EK-10）亦有专门测试。权威与智能分离的判断成立。

### 3.2 原 Archaeology 最重要的 3 个错误
1. **Flow F-05 / EK-07 管道顺序错位（Flow Error）**：真实顺序是 `collapse-check → pre-execute → approval(ask) → guards → execute`；原报告把 approval 放在 guards 之后，且完全遗漏 collapse-check（"确定性失败先于策略管线"这一隐藏契约）。管道顺序是 Authority Flow 的核心，属**实质性 Flow 错误**（非仅措辞）。
2. **guard 覆盖面不完整**：guard 包含 `timeout-policy` 与 `repeat-tool-reminder` 两个成员，原考古只提炼了 timeout-policy（EK-08），遗漏 advisory 型重复调用检测器。
3. **approval 策略模型过度简化**：原 EK-11 称"setApprovalPolicy 单一写路径"（正确），但遗漏 effectivePolicy 的两层解析（config.policy 服务默认 → session override）与 **delegation 播种（子会话策略继承）**——subagent 委派时策略如何继承是 Authority Boundary 的关键机制。

### 3.3 关键遗漏（MISSING）
| # | 遗漏内容 | 证据 | 重要度 |
|---|---|---|---|
| M-1 | **approval 策略委派继承**（override `source:'delegation'` 播种到子会话） | user-approval/src/index.ts:29-35 | 高（Authority Boundary） |
| M-2 | **PTC collapse 隐藏契约**（collapsed call 在策略管线前确定性拒绝） | tools/index.ts:1367-1411 | 高（Hidden Contract） |
| M-3 | **plan-mode per-agent 协作状态**（projection 折叠 + sandbox/approval 独立） | plan/plan-mode/src/index.ts | 高（State/Policy） |
| M-4 | **settings seam 分层解析 + redactSecrets** | settings/settings/src/index.ts | 中 |
| M-5 | **repeat-tool-reminder guard**（advisory） | guard/repeat-tool-reminder/src/index.ts | 中 |
| M-6 | **legacy 格式硬拒绝策略**（兼容=拒绝非支持） | session/src/index.ts:216,362-370 | 中（Failure Path） |
| M-7 | 子系统覆盖缺口扩大：goal / schedule / spill / mcp / feedback / jobs / code-runtime 等 | packages/ 根 | 中（Coverage） |

### 3.4 错误升维
- **无严重错误升维**。KO-03（L4 "层叠安全"）claim 轻微过泛（approval 只覆盖声明 ask 的 gate、sandbox 只覆盖 confined 路径），建议 **DOWNGRADED→scope 收紧**（仍 L4，但 claim 限定"受限执行路径上的分层 fail-closed"）；非 OVER_GENERALIZED（已标 Cross-project pending）。
- KO-04 四层记忆不完整（遗漏 plan projection 实例）但不构成升维错误。

### 3.5 事实错误
- **未发现直接事实错误**。26 条 EK 的 source/symbol 全部真实存在；错误类型均为"不完整/顺序不精确"，而非"断言与代码矛盾"。

### 3.6 Flow 错误
- **F-05（Authority）部分 INCORRECT**：approval 顺序错位（应在 guards 之前）；遗漏 collapse 门（先于整个策略管线）。
- 其余六条流（F-01/02/03/04/06/07）Edge 逐条核实，均真实。

### 3.7 新的 Benchmark / Regression Case（可作 skill CI Gold Record）
| # | Case | 来源 | 验证性质 |
|---|---|---|---|
| B-1 | **never policy 不可绕过**：answerer 在服务 mount 后 PREPENDED 仍无法绕过（fail-closed 不可绕过性） | approval.spec.ts:430 | Regression（权限） |
| B-2 | **守卫单调性**：pre-execute force allow 不能 bypass owner-level monotonic guard | scoped.spec.ts:303 | Regression（权限/管道） |
| B-3 | **collapse 先于策略**：collapsed call 不得被策略监听器观察/批准（隐藏契约） | tools/index.ts:1367 | Regression（管道顺序） |
| B-4 | **plan-mode 恢复**：resume/fork 经 plan projection 恢复协作状态 | plan-mode/src/index.ts | Benchmark（会话投影） |
| B-5 | **legacy 拒绝**：request/header-delta 旧格式与 reason:'fallback' 被硬拒绝 | session/src/index.ts:216,362 | Benchmark（兼容策略） |

---

## 四、独立 Auditor 结论
- 原考古**核心骨架可靠**（25 CONFIRMED，0 CONTRADICTED，无事实错误），成功提炼了项目最独特的三个洞见（日志即真相 / 能力缝 / fail-closed 族）。
- **需要修订**：F-05/EK-07 管道顺序（Flow 级错误）、guard 与 approval 模型不完整、KO-03 scope 收紧。
- **需要补充**：M-1..M-7 遗漏（尤其 delegation 策略继承、PTC collapse、plan-mode）。
- **建议**：以本报告 B-1..B-5 作为 skill 新 Gold Record 素材；修订后的考古结果可在交付 Corpus 前完成。
- 已遵守约束：未修改原 Archaeology Result。
