# 06 — Validation & Evidence

> 独立 Auditor 盲重建：不把考古结果当事实来源，独立重读仓库建立 Independent Findings 后对比。
> 判定：CONFIRMED / PARTIALLY_CONFIRMED / DOWNGRADED / OVER_GENERALIZED / MISSING / CONTRADICTED / NEEDS_HUMAN_REVIEW。
> 禁止修改原考古产物；修正经 Reconciliation 写入 corpus。

## 6.1 盲重建方法
- 独立重读文件集：`specification/a2a.proto`（全文）、`docs/topics/life-of-a-task.md`、`agent-discovery.md`、`streaming-and-async.md`、`extensions.md`、`extension-and-binding-governance.md`、`multi-tenancy.md`、`custom-protocol-bindings.md`、`a2a-and-mcp.md`、`GOVERNANCE.md`、`SECURITY.md`、`CHANGELOG.md`、`adrs/adr-001-protojson-serialization.md`、`.github/workflows/` 目录
- 先独立记录 Findings，再打开考古包（00-05）逐条对比
- 攻击焦点：单案例→Pattern、Pattern→L4、ADR→实现事实、Flow Edge 真实性、bypass/override/exception/alternate/direct call/admin path/fallback/legacy path、Epistemic 混淆

## 6.2 Independent Findings（盲重建先写，未看考古包）

### IF-01 RPC 清单（独立数法）
a2a.proto:19-140 `service A2AService`：SendMessage / SendStreamingMessage / GetTask / ListTasks / CancelTask / SubscribeToTask / CreateTaskPushNotificationConfig / GetTaskPushNotificationConfig / ListTaskPushNotificationConfigs / GetExtendedAgentCard / DeleteTaskPushNotificationConfig = **11 个方法**。

### IF-02 TaskState 分组（独立读 proto:187-208）
枚举值 9 个（含 UNSPECIFIED）。注释明确：COMPLETED/FAILED = terminal；CANCELED = terminal；REJECTED = terminal（"This is a terminal state"）；INPUT_REQUIRED = interrupted；AUTH_REQUIRED = interrupted；SUBMITTED/WORKING 无 terminal/interrupted 标注。**终态 4、中断态 2**。

### IF-03 SubscribeToTask 终态错误（proto:75）
注释原文："Returns `UnsupportedOperationError` if the task is already in a terminal state (completed, failed, canceled, rejected)." —— 与 6 项中的 4 终态枚举一致。

### IF-04 ListTasks 分页（proto:676-713）
page_size："If unspecified, at most 50 tasks will be returned. The minimum value is 1. The maximum value is 100."；page_token 游标；filter：context_id / status / status_timestamp_after；include_artifacts "Defaults to false to reduce payload size"。**默认 50、1~100、include_artifacts 默认 false**。

### IF-05 return_immediately 语义（proto:155-160）
false（默认）→ "the operation MUST wait until the task reaches a terminal (COMPLETED, FAILED, CANCELED, REJECTED) or interrupted (INPUT_REQUIRED, AUTH_REQUIRED) state before returning"；true → 创建后立即返回。**阻塞返回点含中断态**（CHANGELOG #1403 佐证 "Clarify blocking calls return on interrupted states"）。

### IF-06 扩展限制（extensions.md §Limitations）
禁止：改变核心数据结构定义（加字段/删必填）、新增 enum 值。自定义属性 → metadata map。**与"核心冻结"一致**。

### IF-07 扩展激活机制（extensions.md §Extension Activation）
HTTP 头 `A2A-Extensions`（逗号分隔 URI）；agent 激活支持项、忽略不支持项；响应 SHOULD echo 已激活列表；默认不激活。**机制 + 默认值精确**。

### IF-08 扩展安全铁律（extensions.md §Security）
"An extension MUST NOT provide a way to bypass the agent's primary security controls"；required:true 只用于核心功能/安全（如消息签名扩展）；扩展数据视为不可信输入。**与 EK-19 一致**。

### IF-09 推送双侧安全（streaming-and-async.md §Security Considerations）
Server：URL 验证（allowlist/ownership/egress 防 SSRF）+ 按 authentication 方案自证。Client：JWT/JWKS 验签或 HMAC/API key/token + 防重放（timestamp/jti）+ 密钥轮换。**完整 JWT+JWKS 流程示例存在**。

### IF-10 tenant 契约（multi-tenancy.md）
客户端 MUST echo `AgentInterface.tenant`（声明时）；接口未设置 tenant 则字段 MUST 被省略。**客户端义务式表述**。

### IF-11 AgentCard 签名（proto:455-467）
AgentCardSignature{protected REQ, signature REQ, header}，注释 "follows the JSON format of an RFC 7515 JSON Web Signature (JWS)"。CHANGELOG #917（v0.3.0）加 signatures。**JWS 证据机制存在**。

### IF-12 ADR-001（adrs/adr-001 全文）
Decision Makers: TSC；Date 2025-11-18；Chosen: ProtoJSON；负面明确列出 enum SCREAMING_SNAKE_CASE 破坏性变更、roundtrip 损失、迁移成本、"Ugly enums"、字段名妥协（message）；决策可逆（可在规范复制 ProtoJSON 约定）。**决策记录完整、可逆性明确**。

### IF-13 A2A/MCP 分层（a2a-and-mcp.md）
"MCP is vertical. It deepens a single agent." / "A2A is horizontal. It connects agents across that boundary."；"Used together, MCP gives each agent depth, and A2A gives your system reach."；§Representing A2A Agents as MCP Resources 存在。**design-intent 文档**。

### IF-14 TSC 治理（GOVERNANCE.md 全文）
8 席位：Google(Microsoft)Cisco(AWS)Salesforce(ServiceNow)SAP(IBM)；投票 1 seat 1 vote；quorum ≥50%；6 周缺席 = inactive（LFX 考勤）不计 quorum；"GitHub is the source of truth for significant project decisions"。

### IF-15 CI 治理（.github/workflows/ 列出）
11 个 workflow：check-linked-issues / conventional-commits / dispatch-a2a-update / docs / issue-metrics / links / linter / release-please / sort-spelling-allowlist / spelling / stale。**无行为测试 workflow**；规范一致性由 buf + .api-linter.yaml（specification/ 下）承担。

### IF-16 生命周期/细化（life-of-a-task.md）
contextId 分组；三种 agent 类型；终态不可重启 + 同 contextId 新任务；referenceTaskIds；artifact 版本客户端管理 + 服务端复用 artifact-name；并行任务示例（航班/酒店/雪地摩托）。

## 6.3 对比判定

### 判定统计
| 判定 | 条数 |
|---|---|
| CONFIRMED | 16 |
| PARTIALLY_CONFIRMED | 1 |
| DOWNGRADED | 0 |
| OVER_GENERALIZED | 1 |
| MISSING | 2 |
| CONTRADICTED | 0 |
| NEEDS_HUMAN_REVIEW | 0 |
| **合计** | 20 |

### 3 成功（CONFIRMED 代表）
1. **任务状态机 + 不可变性**（考古 KO-01 / EK-01/04 ↔ 盲重建 IF-02/IF-16）：终态 4、中断态 2、终态不可重启、细化=同 contextId 新任务 —— 全部与 proto 注释及 life-of-a-task 原文一致 → **CONFIRMED**
2. **扩展安全不变量**（KO-04 / EK-19 ↔ IF-08）："MUST NOT bypass primary security controls"+required:true 纪律 —— 规范原文逐字一致 → **CONFIRMED**
3. **推送通知双侧安全**（EK-14 ↔ IF-09）：SSRF 防护（server）+ JWT/JWKS/防重放（client）双侧义务 —— 文档流程完整可回溯 → **CONFIRMED**

### 3 审计攻击记录（counterexample 定向攻击，全部攻击失败=原断言成立）
1. **攻击 "Task 不可变性过强，中断态可否继续？"**：盲重建确认 interrupted（INPUT_REQUIRED/AUTH_REQUIRED）≠ terminal，客户端可再发消息继续（life-of-a-task §Task Refinements；proto:158-160 阻塞返回点含中断态）→ 考古 EK-04 精确表述"终态不可重启"，未过度宣称 → 攻击失败，**CONFIRMED**
2. **攻击 "扩展激活是否隐式全开？"**：盲重建确认扩展默认 inactive + A2A-Extensions 头显式 opt-in + 未支持则忽略（IF-07）→ EK-18"默认不激活"精确 → 攻击失败，**CONFIRMED**
3. **攻击 "租户路由是否协议规定了格式？"**：盲重建确认 tenant 为 opaque string、语义完全由 server 定义、协议只规定 MUST echo/MUST omit（IF-10）→ KO-06"路由语义留在实现、义务写进规范"精确 → 攻击失败，**CONFIRMED**

### 1 修正（RECONCILIATION 项，写入 corpus 版本）
- **MISSING-1（快照笔误）**：snapshot_artifact.md 中"service A2AService（9 个 RPC：…实际 11 个方法）"——盲重建独立计数为 11（IF-01）；"9 个"为残留笔误。→ Reconciliation 修正为 11（00/01/04 已正确，仅快照表述需改）

### 1 OVER_GENERALIZED（已主动标注，非事后发现）
- **KO-07（A2A/MCP 分层）**：证据为文档设计意图（a2a-and-mcp.md），非运行时验证；已标 `design-intent` + `Cross-project validation pending`。盲重建 IF-13 确认证据仅文档层 → 维持 DOWNGRADED 边界：作为 L4 design-intent 保留，不入 L5 Methodology 全量（其 L5 已限定在"designing agentic systems"语境）。→ **OVER_GENERALIZED 已隔离**

### 2 MISSING（覆盖缺口，如实记录）
- **MISSING-2 错误映射未入 Atlas**：specification §5.4 Error Code Mappings 存在（custom-protocol-bindings.md 引用 `TaskNotFoundError`/`UnsupportedOperationError`），但考古包未展开错误类型表——协议级错误面是 Flow Atlas 的缺口 → 列入候选与后续 refresh 关注
- **MISSING-3 未精读文档**：docs/topics 剩余（what-is-a2a / key-concepts / enterprise-ready）、docs/blog（announcing-1.0）、docs/roadmap.md、MAINTAINERS.md、CONTRIBUTING.md、docs/sdk/、specification/json/README.md —— 本 run 精读了 11 篇 topics 中的 9 篇 + 全部核心 proto/ADR/治理/CHANGELOG；未精读者不影响本 run 结论方向（非协议核心语义），列为下次 refresh 的增量覆盖清单

### CONTRADICTED：0（无考古断言与仓库内容冲突）

### NEEDS_HUMAN_REVIEW：0

## 6.4 Benchmark case（供 skill CI Gold Record）
- **Benchmark-A（C-07）**：A2A"扩展 metadata 叠加，不改核心类型" vs MCP SEP-2640 Skills over MCP —— 协议生态"能力扩展外置化"趋势的跨协议对比，最适合作为下一轮 benchmark 学习用例
- **Benchmark-B（C-01）**：A2A Task 不可变性 vs hermes-agent（corpus PR#19）session 语义 —— 跨项目"不可变执行单元"验证

## 6.5 Reconciliation 汇总
| 项 | 动作 |
|---|---|
| 快照 RPC 计数笔误（9→11） | 修正 snapshot_artifact.md（写入 corpus 前） |
| KO-07 边界 | 维持 design-intent 标注，不升全量 L5 |
| 错误映射缺口 | 记入 candidates（C-08 新增候选，见 05 文件补充） |
| 未精读文档清单 | 记入 run_metadata.refresh_pending |
| 原考古产物 | 禁止修改（本文件为独立记录，修正走 corpus 版本） |

> 注：05_candidates.md 中未含 C-08（错误映射/错误面研究）——本 Reconciliation 后补入 corpus 版本 05 文件。
