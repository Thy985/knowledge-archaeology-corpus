# Validation & Evidence — deepseek-harness

> Job: `ARCH-2026-09-03-001` | 六类 Validator 全部先 Blind Reconstruction；不为"全绿"弱化标准。
> Repository is the Source of Truth：独立证据与 Evidence Graph 冲突时以仓库原始代码为准。

---

## 一、Evidence 清单（38 条，EV-001…EV-038）

| ID | stage | discovered_by | source | claim 摘要 | strength |
|---|---|---|---|---|---|
| EV-001 | code | code-analyst | `core/agent-loop/src/agent.ts:256-307` | runLoop：turn/start→preStep→step 循环→turn/end，finally 闭合 | S3 |
| EV-002 | code | code-analyst | `core/agent-loop/src/agent.ts:309-370` | step()：buildRequest→llm.stream→assistant/chunk→assistant/message（interrupted/usage） | S3 |
| EV-003 | code | code-analyst | `core/session/src/types.ts:261` | SessionEventMap merge-extensible、append-only、lossless JSON、连续 seq | S3 |
| EV-004 | code+test | test-analyst | `core/session/src/index.ts:790`；`core/session/tests/*.spec.ts` | deriveMessages() 从日志派生模型历史（model-visible means logged 测试锁定） | S4 |
| EV-005 | code | code-analyst | `session-persistence-jsonl/src/format.ts:37,154,234` | JSONL+zstd、encodeSegment 路径防 traversal、provenance range-encode | S3 |
| EV-006 | code | code-analyst | `core/session/src/surface.ts:29-67` | surfaceOp append/replace 标记模型可见面；替换范围阴影 | S3 |
| EV-007 | doc+code | doc-analyst | `docs/architecture.md` §Session log；session-projection | projection 注册单元增量折叠已提交事件，stateOf()/snapshot() | S3 |
| EV-008 | code | code-analyst | agent-loop 注册 turnBoundary projection | turnBoundary 共享投影状态 | S3 |
| EV-009 | code | code-analyst | `core/session/src/types.ts:87` | SESSION_FORMAT_VERSION=0 pinned | S3 |
| EV-010 | code | code-analyst | `storage/storage-sqlite/src/schema.ts:20,60,82` | SCHEMA_VERSION=1 单调；不兼容即拒绝（PRAGMA user_version） | S3 |
| EV-011 | code | code-analyst | `core/tools/src/index.ts:144,167,227` | pre-execute/guard/execute/post-execute 管线 | S3 |
| EV-012 | code | code-analyst | `core/tools/src/index.ts:697-741` | ToolGuard 单调拒绝（guardReason 第一个拒绝）；无 allow | S3 |
| EV-013 | code+test | test-analyst | `guard/timeout-policy/src/index.ts` | TOOL_TIMEOUT 结构化码、协作式 deadline、不竞速；测试锁定 | S4 |
| EV-014 | code | authority-analyst | `interaction/user-approval/src/index.ts:48` | OUTCOMES 闭合 fail-closed；unavailable 兜底 | S3 |
| EV-015 | code | authority-analyst | `user-approval/src/index.ts:92,235,258-264` | setApprovalPolicy 单一写路径；effectivePolicy 重放；never 在 dispatch 前确定性拒绝 | S3 |
| EV-016 | code+test | test-analyst | `user-approval/tests/*.spec.ts` | approval/asked+decided 成对审计事件测试；模型不学习审计细节 | S4 |
| EV-017 | code | authority-analyst | `sandbox/sandbox/src/index.ts:29,62-71` | SandboxMode 三态；per-call 携带 | S3 |
| EV-018 | code | authority-analyst | `sandbox/sandbox/src/index.ts:103` | enforcement full/partial 上报事实；网络不可达可靠失败 | S3 |
| EV-019 | code | authority-analyst | `credentials/credentials/src/index.ts:2,29,117` | CredentialRef 引用而非值；provider 拥有值 | S3 |
| EV-020 | code+test | test-analyst | `credentials/tests/*.spec.ts` | 每操作重解析、空值视为不存在测试 | S4 |
| EV-021 | doc | policy-analyst | `docs/subsystems/permission-presets.md` | presets 打包 sandbox+approval；自身不强制 | S3 |
| EV-022 | failure | failure-analyst | `docs/postmortem/0004-*.md` | landlock partial-ABI 误分类；修复=status-gated fatal evidence | S5 |
| EV-023 | doc+code | doc-analyst | `docs/capability-seams.md`（ctx.llm） | LLM adapter 注册表 | S3 |
| EV-024 | code | code-analyst | `llm/llm` 包 | adapterDefaults/请求级覆盖；replay provider | S3 |
| EV-025 | code | code-analyst | `core/agent-loop/src/agent.ts:394-400` | 失败结构化：LlmError 保留 facts，否则 errorChain+UNKNOWN | S3 |
| EV-026 | code | code-analyst | `core/system-prompt/src/index.ts:2,52` | 有序 section 注册、重名抛错、单 complete section 独占 | S3 |
| EV-027 | code | code-analyst | `compaction/compaction/src/index.ts:120-162` | compactNow 仅 idle；changed-span 竞态拒绝 | S3 |
| EV-028 | code+test | test-analyst | `compaction/command-compact`；tests | compaction/start 持久事件；token accounting | S4 |
| EV-029 | code+test | test-analyst | `core/agent-loop/src/agent.ts:280-302`；tests | max-tokens sticky 测试 | S4 |
| EV-030 | code | code-analyst | `subagent/tool-subagent/src/index.ts:43-67` | ctx.subagents provider 委托；model-facing tool 可配置 | S3 |
| EV-031 | code | code-analyst | `subagent/subagent-acp|fork|spawn` 包 | 多 provider 实现 | S3 |
| EV-032 | code | code-analyst | `packages/hooks/claude-code|codex` | 外部 agent SDK 集成 | S3 |
| EV-033 | doc+code | doc-analyst | `docs/capability-seams.md`（generated） | 自动生成 seam 依赖图 | S3 |
| EV-034 | code | code-analyst | `scripts/gen-doc-graphs.ts` | 图由代码生成，非手维护 | S3 |
| EV-035 | code | code-analyst | `vendor/README.md` | vendored Cordis 9 包、link: 解析 | S3 |
| EV-036 | code | code-analyst | `pnpm-workspace.yaml`；bundle 结构 | bundle=config rows+code；分层覆盖 | S3 |
| EV-037 | code+test | policy-analyst | `runtime-diagnostics/invariants/src/index.ts`；tests | allowlist/blocklist；启动 loud 失败；断言范围受 conventions | S4 |
| EV-038 | code+test | policy-analyst | `scripts/verify-application-entrypoints.ts`；AGENTS.md | 拒绝绕过 dsh 的应用路径；测试锁定 | S4 |

**三角印证汇总**：Supported Facts（≥2 独立源）= EV-004,013,016,020,028,029,037,038（8 条 S4 级）；其余为 S3 单源 code/doc 证据（在 EK 中以 code+doc 双视角交叉）。

---

## 二、Blind Reconstruction（盲重建，先于验证）

**阶段 A（盲，未参考 KO/Flow）——对仓库的独立理解**：
- agent-loop 是"日志边界驱动"：每一步模型请求/工具调用都在 session log 留下 start/end 事件对，回合结果从事件判定，非旁路变量。核心文件 `agent-loop/src/agent.ts` 的 runLoop/step 两个方法。
- 权限是"分层默认拒绝"：tools 的 guard 只有拒绝；approval 的 Answerer 链失败闭合 unavailable；sandbox 有显式 danger-full-access 逃生门；credentials 用 env 引用。approval policy 写进日志（approval/policy 事件）→ 重放可重建。
- 工具/沙箱/凭证/LLM/子代理都是"定义+实现+消费"三件套，`docs/capability-seams.md` 是脚本生成的依赖图。
- 记忆四层：日志（真相）→ surface（模型可见）→ projection（派生状态）→ compaction（summary 替换）。

**阶段 B（对比判定）**：
| KO | 独立理解 | 判定 |
|---|---|---|
| KO-01 日志驱动可恢复循环 | 一致 | **CONFIRMED** |
| KO-02 Capability Seam 三件套 | 一致（四系统独立实例） | **CONFIRMED** |
| KO-03 fail-closed 不变量族 | 一致，但发现 danger-full-access 逃生门 | **PARTIALLY_CONFIRMED**（已条件化） |
| KO-04 四层记忆治理 | 一致 | **CONFIRMED** |
| KO-05 日志即真相 | 一致；compaction 为有损压缩 | **PARTIALLY_CONFIRMED**（C-05 登记） |
| KO-06 策略治理闭环 | 一致 | **CONFIRMED** |
| KO-07 结构化错误族 | 一致；SEARCH_FAILED 曾掩盖错误 | **PARTIALLY_CONFIRMED**（已条件化） |
| KO-08 可插拔治理护栏 | 一致 | **CONFIRMED** |

---

## 三、六类 Validator 报告

### 1. truth-auditor（Source Truth）
- **方法**：对 26 条 EK 逐条盲重建后核验 `Where/What symbol`。
- **结果**：26/26 SUPPORTED（EV 均指向实际 symbol）；0 CONTRADICTED。
- **发现**：EK-14（presets "自身不强制"）为 doc 层 Observation，实现中无强制代码 → 标 PARTIAL，保留 Observation 状态，不升 Fact。**verdict: PASS（1 条降级为 Observation）**

### 2. coverage-auditor（Coverage）
- **方法**：独立重搜 high-density 子系统：core（agent-loop/session/tools/system-prompt）、interaction（approval）、sandbox、credentials、subagent、compaction、guard、storage、llm、invariants、acp、web。
- **结果**：核心子系统全部有 ≥1 EK；覆盖缺失登记：`web/` 前端（TUI/GUI RPC 层）与 `acp/`（ACP 协议桥）未深度考古（本 run 聚焦 Runtime 核心）→ **Critical Knowledge Missing（非 blocker，登记为 v2 范围缺口）**。
- **verdict: PASS（带登记缺口）**

### 3. flow-auditor（Flow → KO 交叉校验）
- **方法**：逐条验证 Flow Edge（F-01…F-07 的 symbol/condition/failure path）。
- **结果**：F-01/F-02/F-03/F-05/F-06/F-07 全部 VERIFIED（symbol 存在、条件吻合）；F-04 VERIFIED（invariants+测试）。
- **KO 回溯**：8 个 KO 的 flow_traceability 全部能映射到具体 Edge。**verdict: PASS**

### 4. abstraction-auditor（Promotion Gate，独立判定）
- 对 8 个 KO 逐一独立判断层数：
  - KO-01..06, KO-08 → L4（解释范围 ≥2 机制/跨层，论证块成立）→ **OK**
  - KO-07 → L3（仅错误机制族，不构成认知模型）→ **OK（未升 L4，符合"停留最合适层"）**
  - 全部 KO 不升 L5：无跨项目证据，统一标 Cross-project validation pending。**verdict: PASS（0 过度升维；0 跨级）**

### 5. counterexample-hunter（反例预算制）
- 对每个 L3+ KO 执行 ≥3 个定向反例攻击（searched_paths：bypass / admin / test-only / edge-ABI / 竞态 / 有损）：
  - KO-03：找到 danger-full-access 与 approval 范围限定 → **反例边界化**，claim 已条件化（C-06）
  - KO-07：找到 SEARCH_FAILED 掩盖史 → **反例确认**，已条件化（C-07）
  - KO-01/05：找到 compaction 有损压缩 → **反例边界化**（C-05）
  - KO-02/04/06/08：0 反例，给出搜索证据（bypass 路径不存在于 seam 声明；四层均有独立实现）→ **none**
- **verdict: PASS（3 个反例全部边界化而非推翻；0 反例者附搜索证据）**

### 6. epistemic-auditor（认知状态诚实性）
- 核对状态谱系：Fact（EK 中代码实证）vs Observation（EK-14/22）vs Principle（KO-03/05 标 Cross-project pending）→ 无 Fact→Principle 跳级；无单例→Pattern 冒充；无项目经验→Law。
- **verdict: PASS**

---

## 四、Reconciler（冲突处理）

| 冲突 | 层 | 处理 |
|---|---|---|
| EK-14 声称 presets 不强制 vs 文档暗示预设即权限 | doc(设计) vs code(实现) | CONDITIONAL：presets 是声明/组合物，强制由 approval+sandbox 执行层承担 |
| 日志可重放 vs compaction 有损 | EK-02/03 vs EK-19 | CONDITIONAL：因果链可重建，逐字不可还原（C-05） |
| 结构化错误可归属 vs SEARCH_FAILED 掩盖 | EK-17 vs EK-15 | CONDITIONAL：归属正确性依赖 status-gated 纪律（C-07） |
| fail-closed vs danger-full-access | EK-10/09/13 vs EK-12 | CONDITIONAL：confined 路径 fail-closed；danger-full-access 显式 opt-in（C-06） |
| AGENTS.md 声称"一切可插拔" vs 启动强制 | EK-24 vs EK-26 | CONDITIONAL：可插拔限于运行时能力；应用启动边界强制 |

---

## 五、质量门总结
| 门 | 结果 |
|---|---|
| Truth Gate | PASS |
| Coverage Gate | PASS（登记 web/acp 缺口） |
| Flow→KO 交叉校验门 | PASS |
| Abstraction Promotion Gate | PASS（0 过度升维，L5 全部 pending） |
| Counterexample Budget | PASS（3 反例边界化，0 反例附证据） |
| Epistemic Gate | PASS |
| EK Graph 质量门 | 游离 0%，平均出边 2.4，聚合覆盖 100%，簇规模 4.3 |

**最终判定**：Accept（进入 Final Package）。所有 blocker 无；发现的冲突全部条件化，不弱化证据标准。
