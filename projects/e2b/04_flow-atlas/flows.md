# 04 Flow Atlas — E2B（七类流，从真实代码导出）

> 每条 Flow 的 Edge 必须可回溯到 symbol / file / condition / state transition。禁止凭架构想象画流程。

## 1. Control Flow（创建沙箱主路径）

```
Sandbox.create(template, opts)                        @ sandbox/index.ts
  → SandboxApi.createSandbox(template, timeoutMs, opts)  @ sandboxApi.ts:1643
    → 生命周期强校验（onTimeout 判别联合 / keepMemory×action / autoResume 交叉）@ :1651-1698
    → buildIamBody(opts.iam)                             @ :1160（空 tokens → undefined）
    → buildNetworkBody(network, iam)                     @ :1135（transform 回调此时求值，validate:true）
    → POST /v2/sandboxes (body: NewSandboxV2)            @ :1733
    → handleApiError(res) → 抛错即止
    → compareVersions(envdVersion,'0.1.0') < 0
        ├─ true → SandboxApi.kill(sandboxId) + throw TemplateError  @ :1743-1748
        └─ false → 返回 {sandboxId, sandboxDomain, envdVersion, envdAccessToken, trafficAccessToken}
  → Sandbox 实例构造（持有令牌 + connectionConfig）     @ sandbox/index.ts
```
**关键 Edge**：`createSandbox` 校验失败 → InvalidArgumentError（本地，无网络请求）；`envdVersion` 门失败 → 已创建的沙箱被 kill（补偿动作，防泄漏）。

## 2. State Flow（沙箱状态机）

```
创建 (POST /v2/sandboxes) → running
running ──pause(keepMemory=true)──→ paused ──connect(onResume:'restore')──→ running（内存恢复）
running ──pause(keepMemory=false)──→ paused ──connect(onResume:'reboot')──→ running（冷启动，memory:false）
running ──timeout 触发 onTimeout:'kill'──→ 终止
running ──timeout 触发 onTimeout:'pause' (+autoResume)──→ paused ──traffic 唤醒──→ running
running ──kill()──→ 终止（debug 模式短路 @ sandboxApi.ts:1233）
```
**关键 Edge（condition）**：
- `pause` 返回 409 → 已暂停（`res.error?.code === 409 → return false` @ sandboxApi.ts:1527）
- `keepMemory:false + autoResume:true` → InvalidArgumentError（@ :1694）
- `connect` reboot → body `memory:false`（@ :1856）；restore → 省略 memory 字段
- 旧控制面（<2026-08-20 runtime）不识 memory 字段 → "drops the field and restores memory while answering as if the request had succeeded"（docstring @ :699-703）——**静默降级陷阱**

## 3. Data Flow（执行数据路径）

```
用户代码 → runCode(code, opts) @ code-interpreter-js/src/sandbox.ts:168
  → POST {jupyterUrl}/execute  body: {code, context_id, language, env_vars}
     headers: E2b-Sandbox-Id / E2b-Sandbox-Port / E2B-Traffic-Access-Token / X-Access-Token
  → 流式响应 readLines(res.body) → parseOutput(execution, chunk, callbacks)
     （stdout/stderr/result/error 分派）@ messaging.ts
  → Execution 对象返回

命令数据：Commands.start(cmd) @ packages/js-sdk/src/sandbox/commands/index.ts
  → /bin/bash -l -c {cmd}（user/cwd/envs 用户可控）
  → gRPC stream Start → ProcessEvent(oneof: start/data/end/keepalive) @ spec/envd/process/process.proto
  → keepalive 50s（KEEPALIVE_PING_INTERVAL_SEC @ connectionConfig.ts:12）
```
**关键 Edge**：runCode 握手期 requestTimeout（默认 30s）→ clearTimeout → bodyTimeout（默认 60s）@ sandbox.ts:218-270；流不可重放 → 429 重试只适用于非流式。

## 4. Evidence Flow（证据/可观测性路径）

```
用户 → getMetrics(sandboxId) @ sandboxApi.ts:1356
  → GET /sandboxes/{sandboxID}/metrics?start&end（JS ms → unix s 换算 @ :1370-1375）
  → 404 → SandboxNotFoundError
  → SandboxMetrics[]（cpu/mem/disk 快照）

调试证据：E2B_USER_AGENT_SOURCE（UA 溯源，regex ^[A-Za-z0-9][A-Za-z0-9._-]{0,31}$ @ connectionConfig.ts:356）
  → requestSource='ci' 时 includeDiagnostics=true（code-interpreter 错误带诊断 @ sandbox.ts:151-153）
  → runCode URL 追加 source= 参数 @ :141-149
```
**关键 Edge**：debug 模式 getMetrics 短路返回 `[]`（不产生证据 @ sandboxApi.ts:1363）。

## 5. Authority Flow（权威链）

```
API key（E2B_API_KEY，控制面认证）
  → 创建/连接成功 → envdAccessToken（X-Access-Token header，envd 通道认证 @ envd/api.ts:217）
  → trafficAccessToken（E2B-Traffic-Access-Token header，流量代理认证 @ code-interpreter/src/sandbox.ts:231）
  → MCP gateway：GATEWAY_ACCESS_TOKEN + /etc/mcp-gateway/.token（沙箱内读回 @ sandbox/index.ts）

IAM 工作量身份：SandboxOpts.iam.tokens.<name>（注册 @ buildIamBody :1160）
  → rules.transform 回调 ctx.iam.tokens.<name> = 字面占位符 ${e2b.identity.tokens.<name>}
  → egress proxy 按请求动态签发（秘密不落地 @ sandboxApi.ts:61-79）

Secret：${e2b.secrets.<name>} 由 runtime 解析（write-only，API 永不返回 @ secret.ts）
```
**关键 Edge（陷阱防护）**：`iam.tokens` Proxy 使 toJSON/then/toString/valueOf 读取即抛（@ iam.ts）；未注册名读取抛 InvalidArgumentError——"a typo would surface as a confusing auth failure at the destination"。

## 6. Memory Flow（状态保留/快照路径）

```
代码执行记忆：runCode 带 context_id → Jupyter 内核会话（变量/import 跨调用保留）
  证据：statefulness.test.ts（x=1 → x+=1 → '2'）

沙箱快照记忆：createSnapshot(sandboxId) @ sandboxApi.ts:1562
  → POST /sandboxes/{sandboxID}/snapshots（创建期间沙箱暂停）
  → SnapshotInfo.snapshotId（可作 Sandbox.create 模板）
  → fork：同一快照一次性捕获，N 个 fork 各自独立成败 @ :1759-1822

暂停记忆：pause(keepMemory) @ :1503
  → keepMemory=true：全内存快照（恢复后进程/连接存活）
  → keepMemory=false：仅文件系统快照（恢复=冷启动，进程/连接丢失）
```
**关键 Edge**：`autoResume` 仅对 keepMemory 快照合法——"filesystem-only snapshot has no memory to restore ... must be resumed explicitly via connect()"（@ :451-456）。

## 7. Policy Flow（治理/策略闭环）

```
治理决策（仓库内策略）：
  AGENTS.md/CLAUDE.md：双 SDK 等效变更 + changeset + TASTE.md 设计原则
    → 强制执行路径：CI（lint/format/typecheck）+ generated-files 校验
    → 规范策略：spec/ 禁止手改，bump pin → make codegen（spec/README.md）

安全策略（运行时）：
  envd.yaml x-internal:true 标记控制面操作
    → 沙箱代理据此生成拒绝列表（控制面操作不得经公共 URL 到达）@ spec/envd/envd.yaml 头部
  network.allowOut/denyOut/rules/egressProxy
    → 管道序：过滤 → 每域 transform → 隧道（宿主层）@ sandboxApi.ts
  egress proxy 不可达 → fail-closed（不发直连兜底）@ sandboxApi.ts:159-161

执行策略：
  updateNetwork 原子替换（省略字段=清除；省略 egressProxy=停隧道）@ sandboxApi.ts:1456-1493
    → 策略变更即时生效（无需重启，"Sets or replaces the proxy on a sandbox that is already running, with no restart"）
```
**关键 Edge（Policy→Enforcement）**：策略在"声明处"（规范标记/配置对象）定义，在"执行处"（代理拒绝列表/网络管道）强制；声明与强制由同一标记（x-internal）自动同步，非各处硬编码。

---

## Flow→KO 交叉校验门（v2 契约）

| L1 事实（KO 关键支撑） | Flow Edge 回溯 | 校验 |
|---|---|---|
| KO-01 占位符按请求签发 | Authority Flow: iam.ts → egress proxy（sandboxApi.ts:61-79） | ✓ |
| KO-02 客户端强校验 | Control Flow: createSandbox 校验块（sandboxApi.ts:1651-1698） | ✓ |
| KO-03 探测后归因 | Control/State: handleEnvdApiFetchError → checkSandboxHealth（envd/api.ts:72-91） | ✓ |
| KO-04 两阶段超时 | Data Flow: runCode 双 timer（sandbox.ts:218-270） | ✓ |
| KO-05 过滤→隧道→fail-closed | Policy Flow: buildNetworkEgress → egressProxy（sandboxApi.ts:1084-1113） | ✓ |
| KO-06 Proxy 陷阱 | Authority Flow: iam.ts RUNTIME_PROBED_PROPS | ✓ |
| KO-07 规范所有权 | Policy Flow: spec/README + copy.bara.sky | ✓ |
| KO-08 可恢复性 | Memory Flow: context_id + systemd 自愈测试 | ✓ |
| KO-09 多连接形态 | Control Flow: getSandboxUrl/getSandboxDirectUrl 分支（connectionConfig.ts:526-561） | ✓ |
