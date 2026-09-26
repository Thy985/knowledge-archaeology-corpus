# 02 Engineering Knowledge — E2B（EK Graph，宽底座）

> v3.1 契约：每条 EK 是图节点，必须声明 `links`（6 类边）；KO 从 EK 图的簇聚合（见 03 层）。
> 证据格式：`symbol @ file:line`（repo 相对路径）。L1/L2 独立保留，不因"不够抽象"删除。

---

## EK-01 沙箱生命周期 API 集
- **claim**：Sandbox 门面把沙箱生命周期的全部操作（create/connect/fork/pause/kill/setTimeout/createSnapshot/listSnapshots/getInfo/getMetrics/updateNetwork）收敛为静态方法集，统一经 ClientFactory.resolveOpts 解析连接选项。
- **category**: ARCHITECTURE · **abstraction**: L1 · **value**: A · **epistemic**: Fact
- **evidence**: `class Sandbox` 各静态方法 @ `packages/js-sdk/src/sandbox/index.ts`；`ClientFactory.resolveOpts` @ `packages/js-sdk/src/connectionConfig.ts:603`
- **links**: [{to:EK-02, type:subsystem, undirected}, {to:EK-25, type:mechanism, undirected, note:"静态方法集 + boundOpts 绑定"}]

## EK-02 createSandbox 客户端强校验（生命周期契约前置）
- **claim**：创建沙箱时，onTimeout 判别联合（'pause'|'kill'|对象形式）在客户端就完成语义校验：keepMemory 仅限 pause、autoResume 仅限 pause、keepMemory:false 禁 autoResume；非法组合抛 InvalidArgumentError，绝不把歧义请求发给服务端。
- **category**: ENGINEERING · **abstraction**: L2 · **value**: A · **epistemic**: Fact
- **evidence**: `createSandbox` 校验块 @ `packages/js-sdk/src/sandbox/sandboxApi.ts:1648-1698`（"The action never reaches the API — it is resolved here into the boolean autoPause"）
- **links**: [{to:EK-01, type:subsystem, undirected}, {to:EK-16, type:mechanism, undirected, note:"契约前置校验：本地校验非法名/非法组合"}, {to:EK-03, type:causal, out, note:"创建成功→envd 版本门"}]

## EK-03 envdVersion 版本门控
- **claim**：创建响应携带 envdVersion；SDK 用 compare-versions 与 0.1.0 比较，旧模板立即 kill 并抛 TemplateError（"update the template to use the new SDK"）——保证新 SDK 只与支持其契约的运行时配对。
- **category**: ENGINEERING · **abstraction**: L2 · **value**: A · **epistemic**: Fact
- **evidence**: `compareVersions(res.data!.envdVersion, '0.1.0') < 0` → kill + TemplateError @ `packages/js-sdk/src/sandbox/sandboxApi.ts:1743-1748`
- **links**: [{to:EK-02, type:causal, in, note:"创建请求→版本门"}, {to:EK-19, type:mechanism, undirected, note:"supportsStdinClose 同为版本门模式"}]

## EK-04 debug 模式本地短路
- **claim**：`E2B_DEBUG=true` 时 SDK 连接 localhost envd（apiUrl=http://localhost:3000，host=localhost:port），且 kill/getMetrics/create 在 debug 下短路（不发请求）——本地测试不产生副作用。
- **category**: TOOLING · **abstraction**: L1 · **value**: B · **epistemic**: Fact
- **evidence**: `if (config.debug) { return true }` @ `packages/js-sdk/src/sandbox/sandboxApi.ts:1233`、getMetrics `return []` @ :1363；`this.debug ? 'http://localhost:3000'` @ `connectionConfig.ts:460`
- **links**: [{to:EK-24, type:contrast, undirected, note:"本地直连 vs 生产域名寻址"}]

## EK-05 令牌三级权威链
- **claim**：控制面用 API key 认证；沙箱建立后 SDK 持有 envdAccessToken（X-Access-Token header）与 trafficAccessToken（E2B-Traffic-Access-Token header）两级沙箱内令牌；MCP gateway 另有 GATEWAY_ACCESS_TOKEN。
- **category**: PERMISSION · **abstraction**: L2 · **value**: A · **epistemic**: Fact
- **evidence**: `'X-Access-Token': config.envdAccessToken` @ `packages/js-sdk/src/envd/api.ts:217`；`'E2B-Traffic-Access-Token'` @ `packages/code-interpreter-js/src/sandbox.ts:231`；mcpGateway @ `packages/js-sdk/src/sandbox/index.ts`
- **links**: [{to:EK-06, type:dependency, out, note:"双通道都依赖令牌"}, {to:EK-13, type:subsystem, undirected, note:"权威子系统"}, {to:EK-18, type:subsystem, undirected}]

## EK-06 envd 双通道（REST + gRPC 流）
- **claim**：沙箱内运行时 envd 通过两套通道暴露：REST（/health、/metrics、/execute 类 HTTP）与 gRPC 流（process/filesystem 的 Start/Connect/WatchDir 等流式 RPC）——命令/文件走流式，健康/指标走 HTTP。
- **category**: ARCHITECTURE · **abstraction**: L2 · **value**: A · **epistemic**: Fact
- **evidence**: `EnvdApiClient` @ `packages/js-sdk/src/envd/api.ts:193`；gRPC 客户端 @ `packages/js-sdk/src/envd/rpc.ts`；proto @ `spec/envd/process/process.proto`（`rpc Start(...) returns (stream StartResponse)`）
- **links**: [{to:EK-05, type:dependency, in}, {to:EK-07, type:subsystem, undirected, note:"envd 通道共享错误归因"}, {to:EK-08, type:causal, out, note:"流式通道→连接治理"}]

## EK-07 健康探测错误归因（死沙箱 vs 瞬断）
- **claim**：envd 调用连接中断（fetch terminated / gRPC 断开）后，SDK 发起 5s 超时的 /health 探测：探测失败(502)→"沙箱被杀/到生命周期终点"→TimeoutError（带说明）；健康→瞬断，抛原错误。这是 SDK 区分"沙箱没了"与"网络抖动"的核心机制。
- **category**: FAILURE · **abstraction**: L2 · **value**: A · **epistemic**: Validated Pattern（测试证实）
- **evidence**: `handleEnvdApiFetchError` @ `packages/js-sdk/src/envd/api.ts:72-91`；`checkSandboxHealth`（HEALTH_CHECK_TIMEOUT_MS=5000，502→false）@ :42-61；测试 `killedSandbox.test.ts`（kill 后 runCode 抛 /sandbox was killed while the request was in progress/）
- **links**: [{to:EK-21, type:mechanism, undirected, note:"gRPC 侧同归因"}, {to:EK-28, type:mechanism, undirected, note:"code-interpreter 侧同归因"}, {to:EK-19, type:dependency, in, note:"命令错误也走此归因"}, {to:EK-32, type:causal, in, note:"测试证实"}]

## EK-08 两阶段超时（握手 → 流生命周期）
- **claim**：流式请求分两阶段控制：握手期（setupRequestController 的 requestTimeout 默认 60s，超时即 abort）成功后 clearStartTimeout，转为 idle-read 超时（只限网络侧：服务端停止发送才触发，慢消费者不触发）；用户 AbortSignal 全程生效。连接释放幂等。
- **category**: ENGINEERING · **abstraction**: L2 · **value**: A · **epistemic**: Fact
- **evidence**: `setupRequestController`/`wrapStreamWithConnectionCleanup`/`createIdleAbort` @ `packages/js-sdk/src/connectionConfig.ts:154-342`
- **links**: [{to:EK-26, type:mechanism, undirected, note:"code-interpreter 同款双阶段超时"}, {to:EK-06, type:causal, in}]

## EK-09 429+Retry-After 重试
- **claim**：控制面请求对 429 且带有效 Retry-After（非负整数秒）做重试（默认 3 次）；请求超时被禁用时 60s 总上限兜底；流式 body 只试一次（不可重放）；Request 输入靠 clone() 安全重放；用户 signal 贯穿。
- **category**: ENGINEERING · **abstraction**: L2 · **value**: A · **epistemic**: Fact
- **evidence**: `withRateLimitRetry` @ `packages/js-sdk/src/retry.ts`；ConnectionOpts.retries docstring @ `connectionConfig.ts:62-72`
- **links**: [{to:EK-10, type:dependency, out, note:"重试并发受信号量约束"}]

## EK-10 FIFO 信号量并发治理
- **claim**：limitConcurrency 用 FIFO 信号量限制并发（连接池默认 100、inflight 默认 1000，环境变量可调、0=关闭）；已知 TODO：slot 在响应头到达即释放而非 body 消费完（注释自陈）——高吞吐代价是连接复用窗口风险。
- **category**: ENGINEERING · **abstraction**: L2 · **value**: B · **epistemic**: Fact（含 TODO 观察）
- **evidence**: `limitConcurrency` @ `packages/js-sdk/src/inflight.ts`（注释明确"TODO: ...release slot as soon as headers arrive"）
- **links**: [{to:EK-09, type:dependency, in}, {to:EK-23, type:mechanism, undirected, note:"输入卫生：env 严格解析"}]

## EK-11 网络三块模型 + updateNetwork 原子替换
- **claim**：沙箱网络配置 = allowOut（默认全放行）/ denyOut / rules（每域 transform）；updateNetwork 是**原子替换**——省略字段即清除，egressProxy 省略即停隧道（docstring 列为陷阱："Repeat it in every update that should keep tunneling"）。allowInternetAccess:false ≡ denyOut:['0.0.0.0/0']。
- **category**: PERMISSION · **abstraction**: L2 · **value**: A · **epistemic**: Fact
- **evidence**: `SandboxNetworkOpts`/`SandboxNetworkUpdate` docstring @ `packages/js-sdk/src/sandbox/sandboxApi.ts:215-387`；`updateNetwork` @ :1466
- **links**: [{to:EK-12, type:causal, out, note:"过滤先于隧道"}, {to:EK-13, type:mechanism, undirected, note:"transform 消费 IAM 占位符"}]

## EK-12 egressProxy SOCKS5 宿主层隧道（fail-closed）
- **claim**：egress 可经 BYO SOCKS5 代理隧道；隧道发生在宿主层（过滤之后），沙箱内代码既看不到代理也无法绕开；DNS 拨号时重解析并 pin 该连接；解析到私有/内网段在沙箱创建前被拒；代理不可达时 **fail-closed**（"outbound connections fail rather than falling back to a direct connection"）；UDP（DNS/QUIC）不隧道。
- **category**: PERMISSION · **abstraction**: L2 · **value**: A · **epistemic**: Fact
- **evidence**: `SandboxEgressProxyOpts` docstring @ `packages/js-sdk/src/sandbox/sandboxApi.ts:148-201`；`buildEgressProxyBody` @ :1056
- **links**: [{to:EK-11, type:causal, in, note:"过滤先于隧道"}, {to:EK-30, type:mechanism, undirected, note:"安全边界：声明/配置化而非各处硬编码"}]

## EK-13 IAM workload identity（占位符按请求签发）
- **claim**：沙箱可注册命名工作量令牌（Secret.iamToken()）；网络 rules transform 回调收到的 `iam.tokens.<name>` 是字面占位符 `${e2b.identity.tokens.<name>}`，由 egress proxy 在转发请求时**动态签发**——秘密本身永不离开平台、SDK 永不见其值；未注册名读取抛 InvalidArgumentError（避免目的地混淆的认证失败）。
- **category**: PERMISSION · **abstraction**: L2 · **value**: A · **epistemic**: Fact
- **evidence**: `SandboxNetworkTransformContext.iam` docstring @ `packages/js-sdk/src/sandbox/sandboxApi.ts:61-79`；`iamTokenPlaceholders` @ `packages/js-sdk/src/sandbox/iam.ts`
- **links**: [{to:EK-14, type:dependency, out, note:"占位符依赖 Proxy 防护"}, {to:EK-11, type:mechanism, in}, {to:EK-05, type:subsystem, undirected}, {to:EK-16, type:constraint, in, note:"名字校验约束占位符语法"}]

## EK-14 IAM Proxy 陷阱防护
- **claim**：iam.tokens 用 Proxy 守卫：RUNTIME_PROBED_PROPS（toJSON/then/toString/valueOf）在**读取即抛**（而非使用时），防 JSON.stringify(callback) 崩溃；`__proto__`/`constructor` 等未注册名不误判；`validate:false` 专给 updateNetwork（无 iam config 无法本地校验）。
- **category**: ENGINEERING · **abstraction**: L2 · **value**: A · **epistemic**: Fact
- **evidence**: `RUNTIME_PROBED_PROPS`、Proxy 逻辑 @ `packages/js-sdk/src/sandbox/iam.ts`；`buildTransformContext([], { validate: false })` @ `sandboxApi.ts:1206`
- **links**: [{to:EK-13, type:dependency, in}, {to:EK-23, type:mechanism, undirected, note:"同族输入卫生/原型污染防护"}]

## EK-15 Secret write-only + 占位符
- **claim**：Secret 值 write-only（API 永不返回）；名内插 `${e2b.secrets.<name>}` 由 runtime 解析；同名校验（非空、无 {}/\p{Cc}）在注册处执行——与 IAM token 名同模式。
- **category**: PERMISSION · **abstraction**: L2 · **value**: A · **epistemic**: Fact
- **evidence**: `validateSecretName` + "Secret values are write-only and never returned" @ `packages/js-sdk/src/secret.ts:5-27`
- **links**: [{to:EK-13, type:subsystem, undirected, note:"权威子系统：占位符模式"}, {to:EK-16, type:mechanism, undirected, note:"同字符集校验"}]

## EK-16 名字字符集校验（占位符语法 + header 双约束）
- **claim**：token 名/secret 名用 `/[{}\p{Cc}]/u` 拒绝花括号与控制字符——因为名字会进入 `${...}` 占位符并被拼进 HTTP header，双重语法约束在注册处一次性完成。
- **category**: ENGINEERING · **abstraction**: L2 · **value**: B · **epistemic**: Fact
- **evidence**: `validateIamTokenName` @ `packages/js-sdk/src/sandbox/iam.ts`；`INVALID_SECRET_NAME_CHARS` @ `packages/js-sdk/src/secret.ts:5`
- **links**: [{to:EK-13, type:constraint, out, note:"约束占位符语法"}, {to:EK-15, type:constraint, out}, {to:EK-02, type:mechanism, undirected, note:"契约前置校验"}]

## EK-17 签名 URL（v1_ 前缀）
- **claim**：沙箱文件上传/下载 URL 由 `sha256(path:operation:user:token[:expiration])` 签名生成，`v1_` 前缀 + base64——公开流量 URL 用签名而非秘密查询参数。
- **category**: PERMISSION · **abstraction**: L2 · **value**: A · **epistemic**: Fact
- **evidence**: `signature.ts`（sha256 公式 + v1_ prefix）@ `packages/js-sdk/src/sandbox/signature.ts`
- **links**: [{to:EK-05, type:dependency, out, note:"签名依赖 token"}, {to:EK-06, type:mechanism, undirected, note:"流量经代理/签名寻址"}]

## EK-18 MCP gateway 集成
- **claim**：mcp 选项 → defaultMcpTemplate（'mcp-gateway'）+ GATEWAY_ACCESS_TOKEN + `/etc/mcp-gateway/.token` 读回；沙箱内 MCP 服务器以 stdio 经 gateway 接入（GitHubMcpServer 支持动态 run/install 命令）。
- **category**: AGENT · **abstraction**: L1 · **value**: B · **epistemic**: Fact
- **evidence**: mcp 相关 @ `packages/js-sdk/src/sandbox/index.ts`（defaultMcpTemplate/GATEWAY_ACCESS_TOKEN）；`GitHubMcpServer` @ `sandboxApi.ts:28-43`
- **links**: [{to:EK-05, type:subsystem, undirected, note:"令牌链延伸"}]（+ EK-02 经 template 关系：MCP 模板也是 template 选项）

## EK-19 Commands 执行（bash -l -c + keepalive + stdin close 版本门）
- **claim**：命令经 `/bin/bash -l -c` 启动（用户可控 user/cwd/envs）；keepalive ping 50s（KEEPALIVE_PING_HEADER）；`supportsStdinClose` 按 envd 版本门控；kill 用 SIGKILL；错误统一走 handleRpcErrorWithHealthCheck 归因。
- **category**: ENGINEERING · **abstraction**: L2 · **value**: A · **epistemic**: Fact
- **evidence**: `Commands.start` @ `packages/js-sdk/src/commands/index.ts`（489 行）；`KEEPALIVE_PING_INTERVAL_SEC=50` @ `connectionConfig.ts:12`
- **links**: [{to:EK-03, type:mechanism, undirected, note:"版本门控模式"}, {to:EK-07, type:dependency, out, note:"错误归因"}, {to:EK-21, type:mechanism, undirected, note:"RPC 错误映射"}]

## EK-20 错误树设计（SandboxError 基类 + 状态码）
- **claim**：全部错误继承 SandboxError（带 statusCode）；TimeoutError 对应 unavailable/canceled/deadline_exceeded/unknown 四种 gRPC 码语义；NotFoundError 拆 FileNotFound/SandboxNotFound；**ServiceBusyError 故意不在 SandboxError 树内**（503 无状态变更，须显式 catch）；RateLimitError=429。
- **category**: ENGINEERING · **abstraction**: L2 · **value**: A · **epistemic**: Fact
- **evidence**: `packages/js-sdk/src/errors.ts`（256 行，各类 docstring）
- **links**: [{to:EK-21, type:subsystem, undirected, note:"错误子系统"}, {to:EK-22, type:subsystem, undirected}, {to:EK-07, type:constraint, out, note:"错误语义约束归因路径"}]

## EK-21 gRPC Code→Error 映射 + 五运行时连接中断匹配
- **claim**：envd gRPC 错误按 Code 映射到错误类；`CONNECTION_TERMINATED_MESSAGES` 覆盖 Node/Bun/Deno/CF Workers/Browser 五档运行时各自的连接中断消息变体；`handleRpcErrorWithHealthCheck` 在盲区做健康探测兜底。
- **category**: ENGINEERING · **abstraction**: L2 · **value**: A · **epistemic**: Fact
- **evidence**: `packages/js-sdk/src/envd/rpc.ts`（182 行）
- **links**: [{to:EK-20, type:subsystem, undirected}, {to:EK-07, type:mechanism, undirected, note:"同归因机制"}, {to:EK-19, type:mechanism, in}]

## EK-22 envd HTTP 错误映射
- **claim**：envd REST 错误映射：400→InvalidArgument、401→Authentication、404→NotFound、429→RateLimit、**502→沙箱超时**、507→NotEnoughSpace；未命中→SandboxError；openapi-fetch 空 body 时按 status 判定（非 error 对象）。
- **category**: ENGINEERING · **abstraction**: L1 · **value**: B · **epistemic**: Fact
- **evidence**: `DEFAULT_ERROR_MAP` @ `packages/js-sdk/src/envd/api.ts:24-32`
- **links**: [{to:EK-20, type:subsystem, undirected}, {to:EK-09, type:mechanism, undirected, note:"429 语义一致"}]

## EK-23 输入卫生（__proto__ 防护 + env 严格解析）
- **claim**：mergeOpts 用 defineProperty 写入合并结果，使 `__proto__` 键（如解析 JSON 所得）成为普通自有属性而非污染原型；环境变量用 parseIntEnv/parsePositiveIntEnv 严格解析，非整数/非正数**大声失败**而非静默回退。
- **category**: ENGINEERING · **abstraction**: L2 · **value**: A · **epistemic**: Fact
- **evidence**: `ConnectionConfig.mergeOpts` @ `connectionConfig.ts:477-500`（"defineProperty so a __proto__ key ... becomes a plain own property"）；`parseIntEnv` @ `packages/js-sdk/src/api/metadata.ts`
- **links**: [{to:EK-14, type:mechanism, undirected, note:"同族输入卫生"}, {to:EK-10, type:mechanism, undirected, note:"env 解析"}]

## EK-24 supportedDomains 白名单寻址
- **claim**：仅 e2b.app/e2b.dev/e2b.pro/e2b-staging.dev 白名单域用稳定的 `sandbox.<domain>` 子域；其余域直连 `${port}-<sandboxId>.<domain>` host；browser runtime 不用 sandbox 子域（CORS 问题待解决）。注释自陈"stable sandbox host is only guaranteed for E2B prod"。
- **category**: ENGINEERING · **abstraction**: L1 · **value**: B · **epistemic**: Fact
- **evidence**: `supportedDomains` + `getSandboxUrl` @ `connectionConfig.ts:7,526-546`
- **links**: [{to:EK-04, type:contrast, undirected, note:"寻址策略对照"}]

## EK-25 boundOpts 静态绑定（多 client 隔离）
- **claim**：E2B client 把 connection opts 绑定到子类静态 `boundOpts`（Sandbox/Volume/Secret/Template 各自），resolveOpts 按"per-call > bound > env"合并——多 client（不同 API key/域）可在同一进程共存且互不串扰。
- **category**: ARCHITECTURE · **abstraction**: L2 · **value**: A · **epistemic**: Fact
- **evidence**: `client.ts`（96 行）+ `ClientFactory.boundOpts` @ `connectionConfig.ts:583-608`
- **links**: [{to:EK-01, type:mechanism, undirected, note:"静态方法 + boundOpts"}]

## EK-26 runCode 双阶段超时 + keepalive
- **claim**：code-interpreter runCode 先 requestTimeout（默认 30s，握手）后 bodyTimeout（默认 60s，执行）；fetch 带 keepalive；错误经 extractError 解析后由 handleRequestError 归因。
- **category**: ENGINEERING · **abstraction**: L2 · **value**: A · **epistemic**: Fact
- **evidence**: `runCode` 双 timer @ `packages/code-interpreter-js/src/sandbox.ts:213-295`；`DEFAULT_TIMEOUT_MS`/`JUPYTER_PORT` @ `packages/code-interpreter-js/src/consts.ts`
- **links**: [{to:EK-08, type:mechanism, undirected, note:"同款两阶段超时"}, {to:EK-27, type:dependency, in, note:"context 由 runCode 消费"}]

## EK-27 context 状态保留（Jupyter 会话）
- **claim**：code-interpreter 用 context_id 维系 Jupyter 会话：变量/import/函数跨 runCode 调用保留（测试 statefulness: x=1 → x+=1 → 2）；context 支持 create/list/remove/restart，语言 + cwd 可选。
- **category**: RUNTIME · **abstraction**: L2 · **value**: A · **epistemic**: Validated Pattern（测试证实）
- **evidence**: `context_id` 请求体 @ `packages/code-interpreter-js/src/sandbox.ts:241-246`；`statefulness.test.ts`（期望 '2'）；`contexts.test.ts`
- **links**: [{to:EK-26, type:dependency, out, note:"runCode 消费 context"}, {to:EK-32, type:causal, in, note:"测试证实"}]

## EK-28 code-interpreter 沙箱被杀归因
- **claim**：runCode 请求连接关闭时先 isRunning() 探测：沙箱确实死了 → 描述性 TimeoutError（含修改 timeoutMs/setTimeout 的修复指引）；探测失败则保守假设沙箱在跑、抛原错误（不误报沙箱消失）。
- **category**: FAILURE · **abstraction**: L2 · **value**: A · **epistemic**: Validated Pattern
- **evidence**: `handleRequestError` @ `packages/code-interpreter-js/src/sandbox.ts:460-475`；`killedSandbox.test.ts`
- **links**: [{to:EK-07, type:mechanism, undirected, note:"同归因机制（探测→归因）"}]

## EK-29 spec 所有权边界（Copybara pin 同步）
- **claim**：spec/ 下 openapi.yml/envd proto/volume-content 由 Copybara 从 e2b-dev/runtime 与 belt 按 pin commit 同步，**禁止手改**；`make codegen` 重新拉取并生成客户端；生成文件 CI 校验与 pin 一致。本仓库是规范消费者而非源头。
- **category**: WORKFLOW · **abstraction**: L1 · **value**: A · **epistemic**: Fact
- **evidence**: `spec/README.md`（"don't edit them by hand"）；`runtime-ref`（f5dc6426）；`copy.bara.sky`
- **links**: [{to:EK-30, type:constraint, out, note:"规范同步约束安全标记来源"}, {to:EK-31, type:dependency, out, note:"双 SDK 依赖同一规范"}]

## EK-30 x-internal 声明式安全边界
- **claim**：envd.yaml 中 `x-internal: true` 标记 orchestrator 控制面操作（经宿主网络直达沙箱 slot IP，绝不通过公共沙箱 URL 到达）；沙箱代理从该标记**生成拒绝列表**——新增控制操作只需打标记即自动对公网关闭，安全边界由声明维护。
- **category**: PERMISSION · **abstraction**: L2 · **value**: A · **epistemic**: Fact
- **evidence**: `spec/envd/envd.yaml` 头部注释（"The sandbox proxy generates its rejection list from this marker"）
- **links**: [{to:EK-29, type:constraint, in}, {to:EK-12, type:mechanism, undirected, note:"安全边界声明化"}]

## EK-31 双 SDK 同步契约
- **claim**：AGENTS.md/CLAUDE.md 强制：SDK 变更须在 JS 与 Python（sync+async）等效实现；public surface 变更需 changeset；CLI/js-sdk/python-sdk 需 examples；遵循 TASTE.md 设计原则。
- **category**: WORKFLOW · **abstraction**: L1 · **value**: B · **epistemic**: Fact
- **evidence**: `CLAUDE.md`（"ensure equivalent changes are applied to both JS as well as sync and async Python implementations"）、`AGENTS.md`
- **links**: [{to:EK-29, type:dependency, in, note:"双实现共享契约"}]

## EK-32 测试揭示的行为（killed/statefulness/systemd/reconnect）
- **claim**：测试体系把不可观测的工程质量变成可执行断言：①沙箱执行中被杀 → 描述性 TimeoutError（killedSandbox.test.ts，用 300s sleep 保证只有 kill 能结束请求）；②状态跨调用保留（statefulness）；③jupyter/code-interpreter 被 kill 后 systemd 自动重启、健康恢复、代码可继续执行；④connect 后会话可重建（reconnect.test.ts）。测试即行为契约。
- **category**: TESTING · **abstraction**: L2 · **value**: A · **epistemic**: Fact
- **evidence**: `packages/code-interpreter-js/tests/{killedSandbox,statefulness,systemd,reconnect}.test.ts`；vitest config（testTimeout 60s、live sandbox）
- **links**: [{to:EK-07, type:causal, out, note:"证实归因行为"}, {to:EK-27, type:causal, out, note:"证实状态保留"}]

---

## EK Graph 质量自检

- EK 数：32；有 links 的 EK：32/32
- 平均出边：约 2.2（每 EK links 1~3 条）
- 游离 EK（未连入任何 KO 簇）：0（全部 32 条至少连入 1 个 KO 簇，见 03 层 derivation.facts 并集）
- 全部 EK 证据可回溯至 `repo/` 内 symbol @ file:line
