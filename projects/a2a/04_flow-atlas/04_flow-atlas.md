# 04 — Flow Atlas（七类流）

> 全部从 `repo/` 真实内容导出；关键 Edge 可回溯 symbol/file/condition/state transition。
> 协议仓库无运行时，Flow 以"规范定义的协议流程"呈现——每条 Edge 标注规范锚点。

## 4.1 Control Flow（控制流）

```
client ──SendMessage(SendMessageRequest{message, configuration})──▶ agent
  │                                                                  │
  │  configuration.return_immediately=false（默认）[P:155-160]        │
  │  ├─ 阻塞直到 terminal|interrupted [P:158-160]                     │
  │  └─ agent 决策：
  │       ├─ 无状态 → 返回 Message（stateless interaction）[T:life §Agent Response]
  │       └─ 有状态 → 创建 Task，后续只返回 Task 对象 [T:life §Task-generating/Hybrid]
  │
  ├─ return_immediately=true → 创建 Task 后立即返回 [P:155-157]
  │
  └─ SendStreamingMessage → SSE 流（HTTP 200 + text/event-stream）[T:stream]
       └─ 任务达 terminal|interrupted → server 关闭流，不再更新 [T:stream]
            └─ 断线 → SubscribeToTask 重订阅（终态任务返回 UnsupportedOperationError）[P:74-75]
```
**关键 Edge**：`return_immediately=false`（默认）→ 等待终态/中断态；`=true` → 立即返回（condition: SendMessageConfiguration.return_immediately，[P:160]）。

## 4.2 State Flow（状态流）

```
TaskState 状态机 [P:187-208]
  UNSPECIFIED(0)
  SUBMITTED(1) ──▶ WORKING(2) ──▶ COMPLETED(3)   [terminal]
       │              │      └──▶ FAILED(4)      [terminal]
       │              │      └──▶ CANCELED(5)    [terminal，CancelTask RPC 触发]
       │              │      └──▶ REJECTED(7)    [terminal，agent 决定不执行]
       │              └──────▶ INPUT_REQUIRED(6) [interrupted，等用户输入]
       │              └──────▶ AUTH_REQUIRED(8)  [interrupted，等认证]
       └── 终态后不可重启（Task Immutability）[T:life §Task Immutability]
            └── 后续 = 同 contextId 新建 Task（refinement 模式）[T:life §Task Refinements]
```
**关键 Edge**：`WORKING→INPUT_REQUIRED`（agent 需澄清，客户端可再发消息继续）；`→AUTH_REQUIRED`（认证中断）；任何态→终态后冻结（transition 不可逆）。
**流终止条件**（streaming/push 共用）：state ∈ {COMPLETED,FAILED,CANCELED,REJECTED,INPUT_REQUIRED} 即终止（[T:stream] "reaches a terminal or interrupted state…closes the stream"）。

## 4.3 Data Flow（数据流）

```
Message（无状态通信）[P:260-277]
  parts[]: text | raw(bytes,base64) | url | data(JSON) [P:224-234]
  metadata / extensions[] / reference_task_ids[]
      │
      ▼
agent 处理 → Task{id, context_id, status, artifacts[], history[], metadata} [P:167-184]
      │
      ├─ 同步：SendMessageResponse{task|message} [P:779-788]
      ├─ 流式：StreamResponse{task|message|status_update|artifact_update} [P:790-803]
      │     artifact 分块：TaskArtifactUpdateEvent{append, last_chunk} [P:307-322]
      │     （append=true 追加到同 id 前序 artifact；last_chunk=true 终块）
      └─ 推送：webhook 收 StreamResponse → 验真 → GetTask 拉全量补全 [T:stream §Client Action]
```
**关键 Edge**：artifact 更新事件用 `append`/`last_chunk` 组装（condition: TaskArtifactUpdateEvent.append/last_chunk，[P:317-321]）；`Part.data` 为任意 JSON Value（object/array/string/number/boolean/null，[P:233]）。

## 4.4 Evidence Flow（证据流）

```
Agent Card 即证据载体 [P:362-399]
  AgentCard{name, description, supported_interfaces, capabilities, security_schemes,
            security_requirements, default_input/output_modes, skills, signatures, icon_url}
      │
      ├─ AgentCardSignature = RFC 7515 JWS（protected/signature/header）[P:455-467]
      │     → 客户端可验证卡片完整性/来源（evidence: signature）
      ├─ 缓存证据：Cache-Control max-age + ETag 条件请求 [T:disco §Caching]
      └─ 分级证据：GetExtendedAgentCard（认证后）[P:122-129]
            → 敏感信息只在认证后披露 [T:disco §Securing Agent Cards]
```
**关键 Edge**：卡片可信度 ← JWS 签名（evidence 通道）；卡片新鲜度 ← ETag/If-None-Match（condition: HTTP caching，[T:disco]）。

## 4.5 Authority Flow（权威流）

```
AgentCard.security_schemes（map<string, SecurityScheme>）[P:382]
  SecurityScheme = oneof {APIKey|HTTPAuth|OAuth2|OIDC|mTLS} [P:504-517]
      │
      ├─ security_requirements（map<scheme, StringList scopes>）[P:495-499]
      │     → 客户端按所需 scope 认证
      ├─ AgentSkill 可携带独立 security_requirements（per-skill 覆盖）[P:451-452]
      ├─ OAuth2 现代化：authorization_code+PKCE / client_credentials / device_code
      │     （implicit/password deprecated）[P:568-580]
      └─ Push 通知：AuthenticationInfo（IANA scheme + credentials）[P:324-332]
            → webhook 验 JWT/JWKS/HMAC/token [T:stream §Client Webhook Receiver Security]

扩展方法必须与核心方法同等级 authn/authz（不得旁路）[T:ext §Security] ← 权威不变量
```
**关键 Edge**：认证强度 ← security_schemes 声明；授权粒度 ← scopes map + per-skill 覆盖；扩展旁路 = 规范禁止（constraint，[T:ext]）。

## 4.6 Memory Flow（记忆流）

```
contextId 会话分组（协议级记忆单元）[P:172-173][T:life §Group Related Interactions]
  ├─ 首次交互：server 生成新 contextId；后续消息带同 contextId 延续
  ├─ 服务端（LLM agent）：用 contextId 管理对话/LLM 上下文状态 [T:life]
  ├─ 客户端可选带 taskId（与 contextId 必须匹配；只带 taskId 则 server 推断 contextId）[P:254-259]
  ├─ Task.history[]（多轮消息留存）[P:179-181]
  │     history_length 控制返回量（0=不带；未设=不限；server MAY 降级）[P:150-154]
  └─ artifact 版本链：客户端持有（协议不追踪 mutation）[T:life §Tracking Artifact Mutation]
        └─ 服务端复用一致 artifact-name 助客户端跟踪 [T:life]
```
**关键 Edge**：记忆权威在 contextId（服务端状态键）；artifact 版本权威在客户端（协议外）；`message.context_id` 与 `task.context_id` 一致性义务（[P:254-259]）。

## 4.7 Policy Flow（治理流 / 第七类流）

```
TSC 决策（8 席位：Google/MS/Cisco/AWS/Salesforce/ServiceNow/SAP/IBM）[GOV:3-14]
  │  consensus-first；vote = 1 seat 1 vote；quorum ≥50%；
  │  缺席 ≥6 周（LFX 考勤）= inactive 不计 quorum [GOV:66-72]
  ▼
规范变更/扩展提案（GitHub issue 为 source of truth）[GOV:82][T:gov §Proposal]
  ▼
扩展/绑定治理生命周期 [T:gov §Lifecycle]
  Proposal → Maintainer 赞助 → experimental-* 仓库 → 成熟（参考实现/文档/采用）
  → TSC 投票（quorum50%+多数）→ official（ext-/cpb- 前缀 + 官方 URI 命名空间）
  → 可选 Promotion to Core（标准规范变更流程）
  ▼
规范落地 → 客户端/服务端 MUST/SHOULD 遵守（RFC 2119）
  ▼
SDK 支持策略（扩展/绑定默认 disabled，需显式 opt-in；SDK 维护者自主）[T:gov §SDK Support]
  ▼
未来决策（新一轮 TSC 提案/迭代）——闭环
```
**关键 Edge**：扩展晋升核心必须"成熟度证据 + 投票"双门（condition: quorum ≥50% + majority）；URI 命名空间是标识符非 URL（`https://a2a-protocol.org/extensions/{name}/v1`，[T:gov §URI namespaces]）；SDK 侧扩展支持不参与协议一致性判定（[T:gov]）。

---

## Flow→KO 交叉校验
| KO | 对应 Flow | 校验结果 |
|----|-----------|---------|
| KO-01 任务不可变性 | State Flow（终态冻结）| ✓ 一致（flow 4.2 与 EK-04 同源）|
| KO-02 扩展治理 | Policy Flow（生命周期闭环）| ✓ 一致 |
| KO-03 安全协商链 | Evidence + Authority Flow | ✓ 一致 |
| KO-04 安全边界 | Authority Flow（扩展旁路禁止）| ✓ 一致 |
| KO-05 三通知机制 | Control Flow（return_immediately 分支）| ✓ 一致 |
| KO-06 多租户 | Data Flow（tenant echo）| ✓ 一致 |
| KO-07 A2A/MCP | 概念分层（docs 设计意图）| ✓（design-intent 标注）|
