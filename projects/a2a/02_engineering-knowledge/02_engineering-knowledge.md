# 02 — Engineering Knowledge（EK Graph）

> 宽底座：推理原材料。每条 EK 声明 `links`（六类边：mechanism/subsystem/causal/dependency/constraint/contrast），证据可回溯。
> 证据缩写：`[P:L]` = specification/a2a.proto 行号；`[SD]` = docs/specification.md；`[T:life]` = docs/topics/life-of-a-task.md；`[T:disco]` = agent-discovery；`[T:stream]` = streaming-and-async；`[T:ext]` = extensions；`[T:gov]` = extension-and-binding-governance；`[T:a2a-mcp]` = a2a-and-mcp；`[T:multi]` = multi-tenancy；`[T:cpb]` = custom-protocol-bindings；`[ADR]` = adrs/adr-001；`[GOV]` = GOVERNANCE.md；`[CH]` = CHANGELOG.md。

## 核心机制

**EK-01 TaskState 8 态显式状态机**（L1/L2）
任务生命周期被编码为 9 个枚举值（含 UNSPECIFIED）：4 终态（COMPLETED/FAILED/CANCELED/REJECTED）+ 2 中断态（INPUT_REQUIRED/AUTH_REQUIRED）+ SUBMITTED/WORKING。中断态把"需要人/需要认证"显式建模为协议状态，而非错误。
- 证据：`[P:187-208]`；终态/中断态语义注释 `[P:194-207]`
- links: {mechanism: EK-05, subsystem: EK-15}

**EK-02 Message-vs-Task 双轨协商**（L2）
无状态 Message 用于即时应答与"先协商范围"；有状态 Task 用于可追踪的长期执行。三种 agent 类型（Message-only / Task-generating / Hybrid）映射同一双轨。Task 一旦创建，后续只能返回 Task 对象。
- 证据：`[T:life §Agent Response]`
- links: {causal: EK-02→EK-01, contrast: EK-02↔EK-01}

**EK-03 contextId 会话分组**（L2）
contextId 把多个 Task 与独立 Message 分组为同一会话/共同目标；服务端（LLM agent）用其管理上下文状态。首次交互由服务端发新 contextId；后续消息带同 contextId 延续。
- 证据：`[T:life §Group Related Interactions]`；`[P:172-173]` context_id 字段
- links: {mechanism: EK-03↔EK-06, subsystem: EK-15}

**EK-04 Task 不可变性（终态不可重启）**（L2）★
终态任务不能重启；任何后续细化必须同 contextId 新建任务。收益三：输入输出干净映射（可追踪）、明确工作单元、实现无歧义。
- 证据：`[T:life §Task Immutability]`；CHANGELOG #770 "non-restartable tasks"
- links: {causal: EK-04→EK-07, constraint: EK-04→EK-06, subsystem: EK-15}

**EK-05 referenceTaskIds 任务引用**（L2）
Message 用 referenceTaskIds 引用前序任务，提示 agent 以原任务结果为基础；agent 可据此推断相关 artifact。**artifact 版本历史由客户端管理，协议不追踪**——服务端复用一致 artifact-name 帮助客户端跟踪。
- 证据：`[T:life §Referencing Previous Artifacts + §Tracking Artifact Mutation]`；`[P:276]` reference_task_ids
- links: {mechanism: EK-05↔EK-01, constraint: EK-04→EK-05, contrast: EK-05↔EK-07}

**EK-06 并行任务模型**（L2）
同 contextId 内可为每条 follow-up 创建独立并行 Task；客户端可跟踪各任务并在前置完成后创建依赖任务（航班→酒店/雪地摩托示例）。
- 证据：`[T:life §Parallel Follow-ups]`
- links: {dependency: EK-06→EK-03, subsystem: EK-15}

**EK-07 artifact 版本所有权在客户端**（L2）
客户端决定"可接受结果"并维护版本链；服务端不追踪 artifact mutation（协议不规定）。细化时服务端复用一致 artifact-name + 新 artifactId。
- 证据：`[T:life §Tracking Artifact Mutation]`
- links: {constraint: EK-04→EK-07, contrast: EK-07↔EK-05}

## 关键实现

**EK-08 tenant 不透明路由 + 客户端 MUST echo**（L2）
AgentInterface.tenant 为不透明字符串（服务器定义语义）；当接口声明 tenant，客户端必须在每个请求的 tenant 字段 echo；未声明则必须省略。
- 证据：`[P:346-351]`；`[T:multi §Body-Based Routing]`（§8.3.2 规范规则）
- links: {mechanism: EK-08↔EK-09, subsystem: EK-10}

**EK-09 多租户三路由方案**（L2）
URL 子路径 / 认证头（bearer claims/API key）/ body tenant 字段——三方案可组合（产品线用 URL、客户用 tenant）；协议不规定路由实现。
- 证据：`[T:multi §Overview]`
- links: {contrast: EK-09 内三方案互相对照, mechanism: EK-09↔EK-08}

**EK-10 AgentCard 声明式能力清单**（L2）★
AgentCard = 数字化名片：身份（name/description/provider/version）+ 接口（url/binding/tenant/version）+ 能力（streaming/push_notifications/extensions/extended_agent_card）+ 技能（AgentSkill，可带独立 security_requirements）+ 安全方案 + 签名。supportedInterfaces 按序、首个优先。
- 证据：`[P:362-399]`；`[T:disco §Role of the Agent Card]`；`[T:cpb]` "entries listed in preference order"
- links: {causal: EK-10→EK-12, mechanism: EK-10↔EK-13, subsystem: EK-08}

**EK-11 SecurityScheme 五类判别联合 + OAuth2 现代化**（L2）
认证方案 = OpenAPI 3.2 Security Scheme Object 的 discriminated union：APIKey/HTTPAuth/OAuth2/OpenIDConnect/mTLS。OAuth2 支持 authorization_code+PKCE / client_credentials / device_code；implicit/password 已 deprecated（要求 TLS）。
- 证据：`[P:504-646]`；CHANGELOG #1303 "modernize oauth 2.0 flows"
- links: {constraint: EK-11→EK-12, mechanism: EK-11↔EK-14}

**EK-12 分级信息披露（extended agent card）**（L2）
敏感 Agent Card 建议用 authenticated extended agent card（GetExtendedAgentCard RPC）+ 动态 out-of-band credentials，不内嵌静态 secret；registry 可选择性披露。扩展卡缓存为 session-scoped。
- 证据：`[P:122-129]` GetExtendedAgentCard；`[T:disco §Securing Agent Cards]`
- links: {causal: EK-11→EK-12, dependency: EK-12→EK-10, constraint: EK-19→EK-12}

**EK-13 发现三策略按环境选择**（L2）
well-known URI（`/.well-known/agent-card.json`，RFC 8615，公开域）/ curated registry（企业/市场，按 skill/tag 查询，规范不规定 registry API）/ direct config（私有/紧耦合）。缓存用 Cache-Control max-age + ETag 条件请求。
- 证据：`[T:disco §Discovery Strategies + §Caching]`；CHANGELOG #841 "agent.json → agent-card.json"
- links: {mechanism: EK-13↔EK-10, contrast: EK-13 三策略互相对照}

**EK-14 推送通知双侧安全纵深**（L2）★
Server 侧：webhook URL 验证（allowlist/ownership verification/egress firewall 防 SSRF）+ 按 authentication 方案自证（Bearer/API key/HMAC/mTLS）。Client 侧：验 JWT/JWKS 签名或 token + 防重放（timestamp 拒旧/jti 单用）+ 密钥轮换。
- 证据：`[T:stream §Security Considerations]`（完整 JWT+JWKS 示例流程）
- links: {causal: EK-14→EK-16, mechanism: EK-14↔EK-11}

## 关键决策

**EK-15 流式终止绑定任务状态机**（L2）
SendStreamingMessage 开 SSE；任务达终态或中断态 → 服务端关闭流、不再发更新；断线重连用 SubscribeToTask。SubscribeToTask 对终态任务返回 UnsupportedOperationError。
- 证据：`[P:74-75]`；`[T:stream §Streaming]`
- links: {dependency: EK-15→EK-01, causal: EK-15→EK-16, subsystem: EK-03}

**EK-16 Push 通知用于断连/长任务场景**（L2）
Push 面向分钟~天级任务、移动/Serverless 客户端；client 提供 TaskPushNotificationConfig（url/token/authentication）；服务端在显著状态变化发 StreamResponse 负载；客户端验真后用 GetTask 拉全量。
- 证据：`[P:469-485]`；`[T:stream §Push Notifications]`
- links: {causal: EK-14→EK-16, contrast: EK-16↔EK-15}

**EK-17 扩展四类型**（L2）
Data-only（Agent Card 增结构信息）/ Profile（覆盖请求响应模式，可用 metadata 增补子状态，如 working+metadata["generating-image"]）/ Method（新 RPC = Extended Skill）/ State Machine（新增状态/转换）。URI 标识 + 自有规范。
- 证据：`[T:ext §Scope of Extensions]`
- links: {mechanism: EK-17↔EK-18, subsystem: EK-20}

**EK-18 扩展激活协商（默认不激活）**（L2）
客户端在 HTTP 头 `A2A-Extensions`（逗号分隔 URI 列表）请求激活；agent 激活支持项、忽略不支持项；响应 echo 已激活列表。扩展默认 inactive，扩展无关客户端获得基线体验。
- 证据：`[T:ext §Extension Activation]`（完整请求/响应示例）
- links: {causal: EK-17→EK-18, constraint: EK-19→EK-18}

**EK-19 扩展安全铁律：不得绕过主安全控制**（L2）★
扩展新增方法必须与核心方法同等级 authn/authz；required:true 是硬依赖（所有客户端），只用于核心/安全功能（如消息签名扩展）；扩展数据全部视为不可信输入。
- 证据：`[T:ext §Security]`
- links: {constraint: EK-19→EK-18, constraint: EK-19→EK-12, mechanism: EK-19↔EK-20}

**EK-20 核心稳定性铁律：扩展禁止改 core 类型**（L2）
扩展禁止改变核心数据结构定义（加字段/删必填字段）与 enum 值；自定义属性放 metadata map。防破坏核心类型校验。
- 证据：`[T:ext §Limitations]`
- links: {constraint: EK-20→EK-17, mechanism: EK-20↔EK-19}

**EK-21 扩展/绑定两级治理生命周期**（L2）
Proposal(issue) → Maintainer 赞助（创建 experimental-* 仓库）→ Experimental（明确非官方）→ Graduation（生产级参考实现 + 文档 + 采用证据 + TSC 投票 quorum50%/多数）→ Official → Promotion to Core（走标准规范变更流程，非所有扩展都适合入核心）。URI 命名空间为标识符，HTTP 访问不预期。
- 证据：`[T:gov §Lifecycle + §URI namespaces]`
- links: {causal: EK-21→EK-20, mechanism: EK-21↔EK-28}

**EK-22 协议绑定抽象分离**（L2）
v1.0.0 大重构将应用协议定义与传输映射分离；标准三绑定 JSON-RPC/gRPC/HTTP+JSON；自定义绑定（WebSocket/MQTT）必须：全操作支持 + 数据模型等价（camelCase、ISO8601 UTC）+ 行为一致 + 错误映射表 + 流式/认证规范。
- 证据：`[CH: b078419]`；`[T:cpb §Requirements]`
- links: {causal: EK-22→EK-24, subsystem: EK-23}

**EK-23 ADR-001 ProtoJSON 决策**（L2）★
JSON 序列化规范性委托 ProtoJSON 标准而非自定规则。正面：标准化/生态库/一致行为/wire-unsafe 规则现成。负面：enum SCREAMING_SNAKE_CASE 破坏性变更、未知字段 roundtrip 损失、迁移成本、proto 关键字字段名妥协（message）。决策可逆（如发现问题可在规范中复制 ProtoJSON 约定）。
- 证据：`[ADR: 全文]`
- links: {causal: EK-23→EK-22, contrast: EK-23 两个 option 对照}

## 测试揭示行为 / 重要配置 / 边界与例外

**EK-24 无测试体系的规范一致性治理**（L2）
协议仓库无行为测试；CI = super-linter/spelling/conventional-commits/links/release-please/docs/stale 治理类 + buf/api-linter 规范一致性。规范一致性 = "测试"。
- 证据：`.github/workflows/` 11 个文件；`specification/.api-linter.yaml`
- links: {contrast: EK-24↔EK-23, mechanism: EK-24↔EK-28}

**EK-25 查询面分页/过滤契约**（L2）
ListTasks：page_size 1~100 默认 50、page_token 游标、status/context_id/status_timestamp_after 过滤、include_artifacts 默认 false 减负。history_length 语义：0=不返回消息，未设=不限制，server MUST NOT 超发但 MAY 降级。
- 证据：`[P:676-713]`；`[P:150-154, 668-673]` history_length
- links: {subsystem: EK-25↔EK-01, dependency: EK-25→EK-03}

**EK-26 return_immediately 阻塞语义**（L2）
SendMessageConfiguration.return_immediately：false（默认）= 必须等待任务达终态或中断态才返回；true = 创建任务后立即返回。中断态也是"阻塞返回点"。
- 证据：`[P:155-160]`；CHANGELOG #1403 "blocking calls return on interrupted states"
- links: {causal: EK-26→EK-01, contrast: EK-26↔EK-15}

**EK-27 消息/事件终态边界（双端义务）**（L2）
流终止/推送触发均以"终态或中断态"为边界；推送负载=StreamResponse（task/message/statusUpdate/artifactUpdate 四选一）——流式与推送共用负载类型。
- 证据：`[P:791-803]` StreamResponse；`[T:stream §Push Notification Payload]`
- links: {mechanism: EK-27↔EK-15, mechanism: EK-27↔EK-16}

**EK-28 TSC 治理机制**（L2）
八席位投票制：共识优先；quorum ≥50%；缺席 ≥6 周（LFX 考勤）= inactive 不计 quorum；GitHub 为重大决策 source of truth；Discord 聊天非正式（ephemeral）。
- 证据：`[GOV: 全文]`
- links: {causal: EK-28→EK-21, mechanism: EK-28↔EK-24}

**EK-29 Agent Card 签名（JWS）**（L2）
AgentCard.signatures = RFC 7515 JWS（protected base64url header + signature + 可选 header）——Agent Card 可被签名，为客户端提供证据验证基础。
- 证据：`[P:455-467]`；CHANGELOG #917 "Add signatures to the AgentCard"
- links: {mechanism: EK-29↔EK-14, constraint: EK-29→EK-10}

---

## EK Graph 统计
- EK 总数：29
- 出边合计：44（平均 1.5 ≥1 ✓）
- 游离 EK：0（全部 ≥1 边 ✓）
- 关键枢纽：EK-10（4 边）/ EK-20（3 边）/ EK-01（3 边）
