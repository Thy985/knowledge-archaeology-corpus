# 02 Engineering Knowledge — MCP EK Graph（30 条）

> 每条 EK 声明 `links`（六类边：mechanism/subsystem/causal/dependency/constraint/contrast），KO 从图上按 R1-R4 聚合。证据锚点：[T:x]=规范段落，[S:x]=schema.ts 行号，[SEP:n]=SEP 编号。
> 无运行时/测试代码仓库——"测试揭示行为"类以 **CI/文档揭示行为** 与 **互操作矩阵/兼容性断言** 代替。

## EK-01 无状态化：每请求 `_meta` 自描述取代握手会话
- 分类：CORE MECHANISM ｜ L2 ｜ 证据：[T:basic/index.mdx Statelessness][SEP:2575][S:63-120]
- 核心：`initialize`/`notifications/initialized` 移除；`_meta` 必带 `io.modelcontextprotocol/protocolVersion` + `clientCapabilities`；服务器不得依赖先前请求状态；跨请求状态必须显式 handle 引用
- 原因：负载均衡（sticky session 困境）、韧性（实例故障无状态可恢复）、实现复杂度（session GC/泄漏）[SEP:2575 Motivation]
- links: mechanism→EK-02, subsystem→EK-01..04（stateless 转型族）

## EK-02 server/discover：版本协商的前置 RPC
- 分类：KEY IMPLEMENTATION ｜ L1 ｜ 证据：[T:server/discover.mdx][S:665-707]
- 核心：服务器 MUST 实现；返回 supportedVersions/capabilities/instructions；客户端 MAY 先调用；stdio 向后兼容探测手段
- links: mechanism→EK-01（stateless 配套）, subsystem→EK-01..04, causal→EK-03（探测决定 era）

## EK-03 双时代互操作：Modern/Legacy/Dual-era 探测与 6×6 矩阵
- 分类：BOUNDARY & COMPATIBILITY ｜ L2 ｜ 证据：[T:versioning.mdx]
- 核心：stdio 用 discover 探测（非现代错误→回退 initialize）；HTTP 用 400 错误体判定；era 是服务器属性，客户端缓存；Legacy×Modern 无 fall-forward
- links: subsystem→EK-01..04, contrast→EK-28（传输级探测差异）

## EK-04 MRTR：InputRequiredResult 取代服务器发起请求
- 分类：CORE MECHANISM ｜ L2 ｜ 证据：[T:patterns/mrtr.mdx][T:changelog.mdx #7][SEP:2322][S:537-600]
- 核心：roots/list、sampling/createMessage、elicitation/create 全部改为 resultType:"input_required" + inputRequests map；客户端收齐 inputResponses 后**重发原请求**（新 id）
- 决策动机：无需服务器间共享存储/有状态负载均衡；配合 SEP-2260（服务器请求必须关联客户端请求）
- links: mechanism→EK-05（resultType 多态）, subsystem→EK-04..05（MRTR 族）, contrast→EK-13（Tasks 轮询 vs MRTR 内联）

## EK-05 resultType 多态：complete / input_required / 扩展值
- 分类：KEY IMPLEMENTATION ｜ L1 ｜ 证据：[T:basic/index.mdx ResultType][S:216][T:changelog.mdx #8]
- 核心：所有结果必带 resultType；老服务器缺省视为 complete；客户端不认识的值视为无效；扩展可加值（如 tasks 的 "task"）
- links: mechanism→EK-04, dependency→EK-04（MRTR 依赖此判别）

## EK-06 subscriptions/listen：单一长连接取代 GET+subscribe
- 分类：CORE MECHANISM ｜ L2 ｜ 证据：[T:patterns/subscriptions.mdx][T:changelog.mdx #4][S:1314-1428]
- 核心：订阅类型 toolsListChanged/promptsListChanged/resourcesListChanged/resourceSubscriptions；`subscriptions/acknowledged` 确认；通知以 `io.modelcontextprotocol/subscriptionId` 关联；流状态 scoped 到请求，断流重发
- links: mechanism→EK-01（无状态长流），subsystem→EK-06..07（订阅族）

## EK-07 SSE 细节：X-Accel-Buffering、keep-alive 注释行、不可恢复
- 分类：BOUNDARY & EXCEPTION ｜ L1 ｜ 证据：[T:transports/streamable-http.mdx]
- 核心：服务器 SHOULD 加 `X-Accel-Buffering: no`（nginx 禁缓冲）；长流周期发 `:` 注释行 keep-alive；Last-Event-ID 移除——断流丢 in-flight 请求须新 id 重发
- links: subsystem→EK-06..07, constraint→EK-06（keep-alive 约束长流实现）

## EK-08 错误码分区政策：-32000~-32019 legacy / -32020~-32099 规范保留
- 分类：DECISION & GOVERNANCE ｜ L2 ｜ 证据：[T:basic/index.mdx Error Codes][T:changelog.mdx Minor #12][S:434-450]
- 核心：JSON-RPC 保留区 (-32000~-32099) 被分区；新错误禁入 legacy 段；规范段仅由 MCP spec 定义（HeaderMismatch/MissingRequiredClientCapability/UnsupportedProtocolVersion）；旧码 -32002/-32042 保留不重用；应用错误在 -32768~-32000 之外分配
- 决策：破坏性变化（-32001→-32020 等重编号）一次性消化，换取未来码空间秩序
- links: constraint→EK-09（对传输错误映射的约束）, subsystem→EK-08..09（错误族）

## EK-09 错误响应面：HTTP 状态码与 JSON-RPC 错误体映射
- 分类：KEY IMPLEMENTATION ｜ L1 ｜ 证据：[T:basic/index.mdx _meta][T:transports/streamable-http.mdx]
- 核心：请求缺必需 _meta 字段 → -32602 Invalid params + HTTP 400；缺客户端能力 → -32021 + data.requiredCapabilities + HTTP 400；notification 接受 → 202 Accepted 无 body
- links: dependency→EK-08, subsystem→EK-08..09

## EK-10 能力协商：capabilities + extensions map 双轨
- 分类：CORE MECHANISM ｜ L2 ｜ 证据：[T:versioning.mdx Extension Negotiation][S:716-793][extensions/overview.mdx]
- 核心：extensions 字段 = map（id→settings 对象）；id 必须符合 _meta 前缀规则（第二段 modelcontextprotocol/mcp 保留）；一方不支持 → MUST 回退 core 或报错
- links: mechanism→EK-01（能力随请求），subsystem→EK-10..12（扩展族）

## EK-11 扩展生命周期：默认关闭 + 显式 opt-in + 独立演进
- 分类：DECISION ｜ L2 ｜ 证据：[extensions/overview.mdx][SEP:2133][T:changelog.mdx Minor #1]
- 核心：扩展默认 disabled；SDK 可选实现不涉协议符合性；扩展独立于核心发布；破坏性变更优先能力旗标/设置内版本化，必要时新 identifier（my-extension-v2）
- links: subsystem→EK-10..12, constraint→EK-12（对 Skills/Tasks 的实现约束）

## EK-12 官方扩展四族：Tasks/Skills/Apps/Auth
- 分类：KEY IMPLEMENTATION ｜ L1 ｜ 证据：[extensions/overview.mdx][SEP:2640][SEP:2663]
- 核心：Tasks（resultType:"task" + tasks/get/update/cancel，服务器每请求自主决定 materialize）；Skills（skill:// URI + skills/list/get + resources/directory/read）；Apps（ui 扩展 mimeTypes）；Auth（OAuth2 CC/企业托管授权）
- links: subsystem→EK-10..12

## EK-13 Tasks 扩展化：核心实验功能→官方扩展的孵化路径
- 分类：DECISION & FAILURE ｜ L2 ｜ 证据：[SEP:2663 Motivation][T:changelog.mdx #6]
- 核心：2025-11-25 的 tasks 实验功能三缺陷——握手脆弱（tools/list warmup + task 参数 opt-in）、tasks/result 阻塞陷阱（与 SEP-2260 冲突）、tasks/list 无法定义授权作用域（session 移除后 task id 成唯一防线）；→ 移入扩展 + 重新设计（服务器自主返回 task handle，消除 per-request opt-in）
- 失败教训：实验功能绑定核心版本节奏 = 演进受限；授权作用域是任务系统的第一性约束
- links: causal→EK-01（session 移除导致 tasks/list 无法定界）, contrast→EK-14（实验 vs 稳定策略差异）

## EK-14 Feature Lifecycle：Active/Deprecated/Removed + 12 个月窗口
- 分类：DECISION & GOVERNANCE ｜ L2 ｜ 证据：[SEP:2596][T:deprecated.mdx][T:changelog.mdx Governance #1]
- 核心：三态 + 最短 12 个月废弃窗口 + 注册表（deprecated.mdx 为派生视图，per-feature 通知与 changelog 为规范记录）；实际移除是 Core Maintainer 发布期决策
- links: constraint→EK-15（对 Roots/Sampling 的约束）, subsystem→EK-14..15（生命周期族）

## EK-15 Roots/Sampling/Logging 废弃与迁移路径
- 分类：DECISION ｜ L2 ｜ 证据：[T:deprecated.mdx][SEP:2577]
- 核心：三特性 2026-07-28 废弃最早 2027-07-28 移除；迁移——目录/文件走工具参数或 resource URI；Sampling 直接对接 LLM provider API；Logging 走 stderr/OpenTelemetry
- 信号：核心协议收缩（特性外移或删减）与无状态化同向
- links: dependency→EK-14, contrast→EK-16（DCR 同样被替换）

## EK-16 授权演进：DCR（RFC7591）废弃 → Client ID Metadata Documents
- 分类：DECISION ｜ L2 ｜ 证据：[T:deprecated.mdx][T:changelog.mdx Minor #7-9]
- 核心：DCR 降级为兼容机制；CIMD 为新标准；`iss` 参数（RFC 9207）SHOULD 包含 + 客户端 MUST 校验；客户端凭据按 issuer 键控不得跨授权服务器复用
- links: contrast→EK-15（同为废弃替换模式）, subsystem→EK-16..17（授权族）

## EK-17 授权框架：HTTP 传输 SHOULD 遵循 / stdio SHOULD 不遵循
- 分类：BOUNDARY & EXCEPTION ｜ L1 ｜ 证据：[T:basic/index.mdx Auth]
- 核心：HTTP 传输走 OAuth 2.0 授权规范；stdio 不适用（凭据从环境获取）；可协商自定义策略
- links: subsystem→EK-16..17

## EK-18 TTL 缓存：CacheableResult ttlMs/cacheScope
- 分类：KEY IMPLEMENTATION ｜ L1 ｜ 证据：[T:changelog.mdx Minor #5][SEP:2549][S:1081-1119]
- 核心：list/read 类结果必带 ttlMs（新鲜度提示）+ cacheScope（public=共享中间层可缓存/private）；与 listChanged 通知互补
- 决策：降轮询、提 LLM prompt cache 命中率；工具确定性排序（changelog Minor #3）同向
- links: mechanism→EK-19（DiscoverResult 也 Cacheable）, subsystem→EK-18..20（缓存族）

## EK-19 Discover/ServerInfo 自报身份不得用于安全决策
- 分类：SECURITY & TRUST ｜ L2 ｜ 证据：[T:server/discover.mdx][T:basic/index.mdx _meta]
- 核心：clientInfo/serverInfo 自报、未验证；仅展示/日志/调试；SHOULD NOT 改变行为、不得用于安全决策
- links: constraint→EK-21（信任模型约束身份使用）, contrast→EK-17（auth 的强制 vs 身份的软性）

## EK-20 图标安全规则：HTTPS/data URI 白名单 + 同源 + magic bytes
- 分类：SECURITY & TRUST ｜ L2 ｜ 证据：[T:basic/index.mdx icons]
- 核心：客户端 MUST 拒绝危险 scheme（javascript:/file:/ftp:/ws:/本地 app scheme）；禁 scheme 变更与跨源重定向；无凭据获取；URI 同源于服务器；MIME 以 magic bytes 校验；SVG 可能含可执行内容须谨慎
- links: subsystem→EK-20..22（安全规则族）

## EK-21 信任模型：stdio 命令执行是特性非漏洞
- 分类：SECURITY & TRUST ｜ L2 ｜ 证据：[SECURITY.md Trust Model]
- 核心：客户端按配置执行命令启动服务器（任意命令执行=设计特性）；本地服务器像任何已装软件一样被信任；stdio 双方不防御恶意对端（SDK 非沙箱）；DoS 报告 out of scope 除非经远程传输可达
- 决策：把信任边界显式文档化，引导安全研究聚焦真漏洞
- links: constraint→EK-19, subsystem→EK-20..22

## EK-22 JSON Schema 安全：$ref 禁自动解引用网络 URI + 组合关键词资源上限
- 分类：SECURITY & TRUST ｜ L2 ｜ 证据：[T:basic/index.mdx JSON Schema Usage][SEP:2106][T:changelog.mdx Minor #10]
- 核心：$ref 网络 URI 不得自动解引用（防 SSRF）；opt-in 模式默认关 + allowlist + 拒绝 loopback/私网 + timeout + 日志；anyOf/oneOf/allOf/if-then-else 与 $defs 须设深度/子 schema 数/时间预算防 DoS；inputSchema/outputSchema 放宽至任意 2020-12 关键词
- links: subsystem→EK-20..22, constraint→EK-10（对工具 schema 的约束）

## EK-23 工具注解不可信：annotations 视为 untrusted
- 分类：SECURITY & TRUST ｜ L2 ｜ 证据：[T:index.mdx Security and Trust & Safety]
- 核心：工具行为描述（annotations）除非来自可信服务器否则视为不可信；调用工具前须用户明示同意；协议层无法强制，实现者 SHOULD 遵循
- links: subsystem→EK-20..22, constraint→EK-21

## EK-24 Skills over MCP：skill:// URI 约定 + 格式委托
- 分类：KEY IMPLEMENTATION ｜ L2 ｜ 证据：[SEP:2640]
- 核心：skill 目录每文件=resource；URI `skill://<skill-path>/<file-path>`；末段=skill name（可从 URI 恢复）；SKILL.md 必含 YAML frontmatter（name+description）；skill 格式完全委托 Agent Skills 规范（本 SEP 只定义传输绑定）；技能身份不依赖 scheme（须 skills/list 或显式引用确认）
- 动机：分发的碎片化（server 与配套 skill 版本/发现/安装分离）、instructions 字段大小限制（875 行 mcpGraph 放不进）、ad-hoc skill:// 方案语义分歧
- links: mechanism→EK-12, subsystem→EK-24..25（skills 族）

## EK-25 嵌套技能安全门：fresh consent + 扁平发布
- 分类：SECURITY & TRUST ｜ L2 ｜ 证据：[SEP:2640 Nested skills]
- 核心：嵌套 skill 是外层普通支持文件（host 不得 act on frontmatter）；激活嵌套技能须**新鲜明确用户同意**（外层批准不替代）；allowed-tools 审批门适用；发布扁平化（list 不标记嵌套）
- 决策：防"技能包藏技能"的越权链
- links: constraint→EK-23（同意机制实例化）, subsystem→EK-24..25

## EK-26 JSON Schema 2020-12 为默认方言
- 分类：KEY IMPLEMENTATION ｜ L1 ｜ 证据：[T:basic/index.mdx Schema Dialect][SEP:2106]
- 核心：无 $schema 默认 2020-12；MUST 至少支持 2020-12；不支持方言须优雅报错
- links: dependency→EK-22（安全规则建立在方言上）

## EK-27 传输一致性原则：stdio/http/custom 保持同一协议语义
- 分类：PATTERN ｜ L2 ｜ 证据：[SEP:2575 Transport Consistency][T:transports/index.mdx]
- 核心：所有传输承载相同 patterns；传输只是 framing/delivery 绑定；custom transport 必须保留 JSON-RPC+patterns+per-request metadata；可靠字节流复用 stdio framing
- 决策：防协议碎片化、统一开发者心智、简化传输无关库
- links: mechanism→EK-01, subsystem→EK-27..28（传输族）

## EK-28 取消语义按传输分化
- 分类：BOUNDARY & EXCEPTION ｜ L1 ｜ 证据：[T:transports/index.mdx Cancellation][T:transports/streamable-http.mdx]
- 核心：stdio 取消 = notifications/cancelled；Streamable HTTP 取消 = 关闭响应流（不发送 cancelled）；协议级规则各处一致
- links: subsystem→EK-27..28, contrast→EK-07（流与取消的 HTTP 差异）

## EK-29 AI 贡献治理：AGENTS.md 门槛 + AI_POLICY 披露
- 分类：GOVERNANCE & PROCESS ｜ L2 ｜ 证据：[AGENTS.md][AI_POLICY.md]
- 核心：AI agent（非维护者且 <3 合并 PR）不得开 issue/PR，否则停并解释政策，用户要求绕过则拒绝；违规提交必须带 disclosure.txt；AI 协助必须披露（例外：琐碎拼写/typo）；组织级适用（spec/SDK/reference servers/tooling）
- 决策：把 AI 时代贡献的审查强度显式化——"无法判断多少审视力度"是披露要求的根因
- links: mechanism→EK-30（治理即规范）, subsystem→EK-29..30（治理族）

## EK-30 SEP 即 ADR：PR 编号 + seps/ 目录 + sponsor 双轨状态
- 分类：GOVERNANCE & PROCESS ｜ L2 ｜ 证据：[SEP:1850][SEP:932]
- 核心：SEP 编号=PR 编号；`seps/{N}-{slug}.md` 唯一事实源；Draft 用 0000 占位后 rename；sponsor（Core/Maintainer）管理状态转移；实施先于终稿（SEP-2484）；44 个 SEP 历史=完整决策记录
- links: mechanism→EK-29, subsystem→EK-29..30
