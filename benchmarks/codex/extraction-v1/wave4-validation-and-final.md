# Wave 4 · Validation Report + Final Knowledge Package

> 6 类 Validator 交叉验证 + Reconciler 冲突调和 + 最终交付。
> 时间：2026-09-03 | 项目：openai/codex

---

## 一、Validation Report（6 类 Validator）

### 1. truth-auditor（事实核验）

| KO | claim 核验 | 证据支撑 | verdict |
|----|-----------|---------|---------|
| KO-01 | Intelligence ≠ Authority | orchestrator.rs 三态审批 + guardian 独立 session + 沙箱独立 Gate | PASS |
| KO-02 | 声明 ≠ 证据 | guardian transcript token 限制 + 结构化 JSON + 90s 超时 + 熔断器 | PASS |
| KO-03 | 自举递归与逃逸阀 | AGENTS.md:8-10 明确禁止修改 + 环境变量由运行时设置 | PASS |
| KO-04 | 单一真相源 | session.rs:96-99 三字段同步 + 派生函数 | PASS |
| KO-05 | Fail Closed | guardian fail closed + 沙箱升级严格条件 + 附件策略不可绕过 + 多 Agent 超限拒绝 | PASS |
| KO-06 | 核心模块反膨胀 | AGENTS.md:72-83 明确规定 + core 472 文件事实 | PASS |
| KO-07 | 五段式架构指纹 | 4 处独立出现（工具/Guardian/网络/Session） | PASS |

**truth-auditor 总结**：7/7 PASS。所有 claim 均有仓库代码或文档的直接证据支撑，无凭常识或经验的断言。

### 2. coverage-auditor（覆盖审计 · 独立重搜）

coverage-auditor 从 Repository Map 的 A/B 级路径独立重搜，发现以下潜在盲区：

| 盲区 | 路径 | 严重度 | 建议 |
|------|------|--------|------|
| MCP 协议实现细节 | codex-rs/mcp-server/, codex-rs/codex-mcp/ | medium | 本次考古聚焦审批/沙箱/Guardian，MCP 协议的具体实现未深入 |
| 模型提供层 | codex-rs/model-provider/, codex-rs/models-manager/ | medium | 模型切换/降级/重试逻辑未深入 |
| TUI 渲染层 | codex-rs/tui/ | low | TUI 是展示层，非核心认知，本次未深入 |
| 上下文压缩算法 | codex-rs/core/src/compact*.rs | medium | AutoCompact 的具体算法未深入，仅在 Flow 中提及 |
| 记忆系统 | codex-rs/memories/, codex-rs/history/ | medium | Memory Flow 仅基于 Session 状态推断，未深入 memories crate |
| app-server JSON-RPC API | codex-rs/app-server/, app-server-protocol/ | low | 本次聚焦 CLI 模式，app-server 未深入 |

**coverage-auditor 总结**：6 个潜在盲区，其中 4 个 medium。这些盲区不影响已产出的 7 个 KO 的有效性（均有充分证据），但建议在后续深入考古中补充。**verdict: PASS（核心区域覆盖充分，边缘区域有 documented 盲区）**。

### 3. flow-auditor（流审计 · 抽验 symbol）

| Flow | 抽验 symbol | 代码验证 | verdict |
|------|------------|---------|---------|
| F-01 Control | ToolOrchestrator::run (orchestrator.rs:125), request_approval (orchestrator.rs:193), SandboxManager.select_initial (orchestrator.rs:274) | 全部存在，逻辑与描述一致 | PASS |
| F-02 State | Session::new (session.rs:639), SessionConfiguredEvent (session.rs:1531), ForkPersistence (session.rs:75) | 全部存在 | PASS |
| F-03 Data | PermissionProfileState (session.rs:99), file_system_sandbox_policy (session.rs:223), ThreadConfigSnapshot (session.rs:237) | 全部存在 | PASS |
| F-04 Evidence | GuardianReviewContext (guardian/mod.rs:87), GuardianAssessment (guardian/mod.rs:162), GuardianRejectionCircuitBreaker (guardian/mod.rs:170), GUARDIAN_REVIEW_TIMEOUT=90s (guardian/mod.rs:62) | 全部存在 | PASS |
| F-05 Authority | begin_network_approval (orchestrator.rs:66), ActiveNetworkApproval (orchestrator.rs:15), DeferredNetworkApproval (orchestrator.rs:46) | 全部存在 | PASS |
| F-06 Memory | RolloutItem (session.rs:1356), ThreadStore (session.rs:667), AutoCompactWindow (session.rs:796), RealtimeConversationManager (session.rs:67) | 全部存在 | PASS |

**flow-auditor 总结**：6/6 PASS。所有 Flow 中的 symbol 均在代码中真实存在，无概念箭头。每条流的 Gate 逻辑与代码实现一致。

### 4. abstraction-auditor（升维审计）

| KO | 当前层级 | 解释范围扩大是否合理 | 证据支撑 | verdict |
|----|---------|-------------------|---------|---------|
| KO-01 | L4 | 从 codex 审批架构 → "判断力与执行权分离"通用原则，合理（同类：lint/CI/human-in-the-loop） | 5 条证据，多源支撑 | OK |
| KO-02 | L4 | 从 Guardian 证据链 → "声明≠证据"通用原则，合理（同类：TDD/CI 门禁） | 4 条证据 | OK |
| KO-03 | L3 | 从 codex 沙箱自举 → "自举系统逃逸阀"模式，合理（同类：编译器 stage0/VM host-guest/DinD），但仅 3 条证据，未升 L4 | 3 条证据 | OK（L3 合理，未过度升维） |
| KO-04 | L3 | 从 PermissionProfile → "单一真相源"模式，合理（同类：DB schema/K8s desired state/React state） | 2 条证据 | OK |
| KO-05 | L4 | 从 5 处 fail closed → "AI 系统安全默认"原则，合理（同类：防火墙/权限系统/加密默认），标 Principle（非 Law，因未跨项目验证） | 5 条证据 | OK（诚实标注 Principle，未夸大为 Law） |
| KO-06 | L3 | 从 core 反膨胀 → "大型代码库模块治理"模式，合理（同类：微服务/monorepo/Linux 子系统） | 2 条证据 | OK |
| KO-07 | L4 | 从 4 处五段式 → "可审计代理系统架构指纹"，合理（同类：网络协议栈/DB WAL/Web MVC） | 4 处独立出现 | OK |

**abstraction-auditor 总结**：7/7 OK。无过度升维。所有 L4 均有同类对照和多源证据。KO-05 诚实标注为 Principle（非 Law），符合跨项目验证规则。

### 5. counterexample-hunter（反例猎手）

对每个 L3+ KO 搜索"哪里不是这样"：

| KO | 搜索路径 | 反例发现 | 结果 |
|----|---------|---------|------|
| KO-01 | 是否有工具不经审批直接执行？ | Skip 审批策略下工具可直接执行，但这是显式配置的 Full Access 模式，非绕过；strict_auto_review 下即使 Skip 也需 Guardian 审查 | 无有效反例（Skip 是设计内的授权，非 bypass） |
| KO-02 | 是否有审批无证据链？ | Never 审批策略下无 Guardian 审查，但这是用户显式选择的全自动模式，且仍有沙箱 Gate | 无有效反例（Never 是显式授权，非无证据） |
| KO-03 | 是否有沙箱功能可在沙箱内运行？ | 沙箱配置（非 spawn 沙箱本身）可在沙箱内运行；只有需要 spawn Seatbelt 的集成测试需逃逸阀 | 部分反例：逃逸阀仅适用于"沙箱内 spawn 沙箱"的递归场景，非所有沙箱功能。KO-03 已限定为"自举递归"场景，反例不成立 |
| KO-04 | 是否有策略不从 PermissionProfile 派生？ | 环境级配置可覆盖 session 级 PermissionProfile，但这是显式的层级覆盖，仍从配置派生；executor-managed 沙箱保持符号化但最终由执行器应用 | 无有效反例（层级覆盖是设计内的，非独立配置） |
| KO-05 | 是否有安全关键路径 fail open？ | 未发现。所有 Gate 在不确定时均默认拒绝；网络延迟确认在执行失败时立即 finalize（不保留开放状态） | 无反例发现 |
| KO-06 | 是否有功能被有意添加到 core？ | Session 核心状态容器必须在 core（因被所有模块依赖）；AGENTS.md 说"resist"而非"forbid"，允许合理添加 | 无有效反例（core 保留核心状态是合理的，反膨胀是抵制非必要添加） |
| KO-07 | 是否有执行链不遵循五段式？ | 简单的内部函数调用（如工具内部的 helper）不遵循五段式，但这些不是"Agent 执行链"，是内部实现细节 | 无有效反例（五段式适用于 Agent 执行链，非所有函数调用） |

**counterexample-hunter 总结**：7/7 无有效反例。所有潜在反例均为设计内的显式配置或层级覆盖，非真正的 bypass 或矛盾。**未发现反例**（非"普遍成立"，仅"在本次考古范围内未发现反例"）。

### 6. epistemic-auditor（认知状态审计）

| KO | 标注状态 | 证据强度 | 状态匹配 | verdict |
|----|---------|---------|---------|---------|
| KO-01 | Validated Pattern | S4（多源：code+authority+test+failure） | 匹配（≥2 独立源，S4+） | OK |
| KO-02 | Validated Pattern | S4 | 匹配 | OK |
| KO-03 | Validated Pattern | S3-S4 | 匹配 | OK |
| KO-04 | Validated Pattern | S3 | 匹配（S3 已实现，多源） | OK |
| KO-05 | Principle（Cross-project validation pending） | S4-S5 | 匹配（S5 运行时验证，未跨项目，诚实标 pending） | OK |
| KO-06 | Validated Pattern | S3 | 匹配 | OK |
| KO-07 | Validated Pattern | S4（4 处独立出现） | 匹配 | OK |

**epistemic-auditor 总结**：7/7 OK。无 OVERCLAIMED。KO-05 诚实标注为 Principle 而非 Law，符合跨项目验证规则。所有 Hypothesis 均已升级为 Validated Pattern 或 Principle，无未验证的 Hypothesis 混入知识层。

---

## 二、Reconciler（冲突调和）

### 冲突检测结果

本次考古中未发现 doc-analyst 的 design_intent 与 code-analyst 的 implementation 之间的直接冲突。AGENTS.md 中规定的所有工程规则在代码中均有对应实现：

| 设计意图（doc） | 实现事实（code） | 一致性 |
|-----------------|------------------|--------|
| 禁止修改 CODEX_SANDBOX* 环境变量 | 代码中只读使用，无写入 | 一致 |
| core 反膨胀 | Session 是核心容器（合理），非核心功能在独立 crate | 一致 |
| 模型上下文硬限制 | Session 持有 conversation/state，compact 机制存在 | 一致 |
| 测试独立文件+#[path] | guardian/mod.rs:274 使用 `#[path = "..."] mod tests` | 一致 |
| 集成测试优先 | core/suite/ 存在，AGENTS.md 规定 Agent 变更必须加集成测试 | 一致 |
| insta snapshot 测试 | tui 使用 insta，AGENTS.md 规定 UI 变更必须更新 snapshot | 一致 |

### 条件化知识

虽然无直接冲突，但以下知识需条件化标注：

| KO | 条件 |
|----|------|
| KO-01 | "Intelligence ≠ Authority" 适用于拥有自主执行能力的 Agent；纯建议型 AI（无执行权）不适用 |
| KO-05 | "Fail Closed" 适用于安全关键路径；非安全路径（如 UI 渲染）可 fail open |
| KO-07 | "五段式"适用于需要可审计性的代理执行链；一次性脚本不适用 |

**Reconciler 总结**：无未解决冲突。3 条知识已条件化标注 scope。**verdict: PASS**。

---

## 三、Quality Gate（质量门）

| Validator | 结果 |
|-----------|------|
| truth-auditor | 7/7 PASS |
| coverage-auditor | PASS（4 个 documented 盲区，不影响核心结论） |
| flow-auditor | 6/6 PASS |
| abstraction-auditor | 7/7 OK |
| counterexample-hunter | 7/7 无有效反例 |
| epistemic-auditor | 7/7 OK |
| reconciler | PASS（无冲突，3 条条件化） |

**质量门判定：ACCEPT ✅**

---

## 四、Final Knowledge Package（最终知识包）

### 项目档案

| 字段 | 值 |
|------|-----|
| 项目 | openai/codex |
| 类型 | Agent 系统 / CLI 工具 / 运行时 |
| 语言 | Rust（112 crates，core 472 源文件） |
| 构建 | Bazel + Cargo 双构建 |
| 考古时间 | 2026-09-03 |
| 考古范围 | 核心运行时（Session/工具执行/审批/沙箱/Guardian/多 Agent） |

### 知识对象清单（7 个，4A + 3B）

| ID | 标题 | 层级 | 价值 | 认知状态 |
|----|------|------|------|---------|
| KO-01 | Intelligence ≠ Authority（判断力与执行权分离） | L4 | A | Validated Pattern |
| KO-02 | 声明 ≠ 证据（AI 审批的证据生产链） | L4 | A | Validated Pattern |
| KO-03 | 自举系统的递归矛盾与显式逃逸阀 | L3 | B | Validated Pattern |
| KO-04 | 单一真相源与派生策略的一致性治理 | L3 | B | Validated Pattern |
| KO-05 | Fail Closed 作为 AI 系统的安全默认 | L4 | A | Principle（跨项目验证 pending） |
| KO-06 | 大型代码库的核心模块反膨胀治理 | L3 | B | Validated Pattern |
| KO-07 | 产生→验证→授权→执行→记录的五段式架构指纹 | L4 | A | Validated Pattern |

### Most Important Finding（最重要发现）

**codex 作为 Agent 系统的核心设计哲学由三个互补的原则构成：**

1. **Intelligence ≠ Authority**（KO-01）：AI 的判断力与执行权必须分离，执行权由 Approval + Sandbox + Network Proxy 三重 Gate 链控制
2. **Fail Closed**（KO-05）：所有安全关键路径在不确定时默认拒绝，Guardian 超时/失败/格式错误均拒绝，沙箱升级有严格前置条件
3. **五段式架构指纹**（KO-07）：工具执行/Guardian 审批/网络访问/Session 初始化均遵循"产生→验证→授权→执行→记录"，这是可审计代理系统的结构性特征

**这三个原则共同回答了"如何构建一个既自主又安全的 AI Agent 系统"这一核心问题。**

### Top 跨项目可迁移模式

| 模式 | 来源 KO | 可迁移场景 |
|------|---------|-----------|
| 审批-沙箱-执行三级 Gate | KO-01 | 任何自主执行系统 |
| AI 审查 AI + 熔断器 | KO-02 | 自动审批/自动决策系统 |
| 自举逃逸阀 | KO-03 | 编译器/VM/容器自举系统 |
| 单一真相源 + 纯函数派生 | KO-04 | 多维度配置/策略系统 |
| Fail Closed 安全默认 | KO-05 | 任何安全关键系统 |
| 核心模块反膨胀治理 | KO-06 | 大型代码库/monorepo |
| 五段式执行链 | KO-07 | 可审计代理/自动化系统 |

### 统计

| 指标 | 值 |
|------|-----|
| Evidence Pack 总数 | 28 |
| Supported Facts | 10 |
| Problems | 6 |
| Decisions | 7 |
| Flows（六类流） | 6 |
| Patterns | 5 |
| Knowledge Objects | 7（4A + 3B） |
| Validator PASS 率 | 7/7（100%） |
| 反例发现 | 0（无有效反例） |
| 冲突 | 0（无未解决冲突） |
| 条件化知识 | 3 |

### 剩余风险与盲区

1. **MCP 协议实现**：本次未深入 mcp-server/codex-mcp，MCP 工具调用的权限模型可能有额外发现
2. **模型提供层**：model-provider 的降级/重试/切换逻辑未深入
3. **上下文压缩算法**：compact* 的具体算法未深入，可能影响 Memory Flow 的准确性
4. **记忆系统**：memories/history crate 未深入，Memory Flow 基于 Session 状态推断
5. **app-server API**：JSON-RPC API 层未深入，可能有额外的权限边界

这些盲区不影响已产出的 7 个 KO 的有效性，建议在后续深入考古中补充。

---

## 五、交付物清单

| 序号 | 文件 | 内容 |
|------|------|------|
| 0 | `00-skill-architecture.md` | 技能架构权威参考文档 |
| 1 | `01-repository-map.md` | 仓库地图（A/B/C 级路径） |
| 2 | `02-evidence-packs/wave1-evidence-packs.md` | Wave 1 Evidence Packs（28 条） |
| 3 | `04-mining-graphs/wave2-mining-graphs.md` | Wave 2 Evidence Graph + Mining Graphs |
| 4 | `05-knowledge-objects/wave3-knowledge-objects.md` | Wave 3 Knowledge Objects（7 个，L1-L5） |
| 5 | `06-validation/wave4-validation-and-final.md` | Wave 4 Validation + Final Package（本文件） |
