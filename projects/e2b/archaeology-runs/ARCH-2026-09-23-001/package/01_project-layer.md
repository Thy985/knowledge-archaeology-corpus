# 01 Project Layer — E2B 项目地图

> 本层只记录项目事实（L0/L1），全部可追溯到 `repo/` 实际路径。详见 `snapshot_artifact.md`（阶段 3 独立产物）。

## 1. 项目定位

E2B 是"运行 AI 生成代码的安全隔离云沙箱"基础设施。README.md L20: "E2B is an open-source infrastructure that allows you to run AI-generated code in secure isolated sandboxes in the cloud." 发行形态 = SDK + CLI（无服务器运行时；服务器为 E2B Cloud / BYOC，契约在 spec/）。

## 2. 架构分层（按仓库证据）

```
┌─ 用户层 ─────────────────────────────────────────────┐
│  code-interpreter-js (runCode/contexts)  cli         │
├─ SDK 核心（js-sdk / python-sdk sync+async）──────────┤
│  sandbox/（生命周期+网络+IAM） envd/（沙箱内通道）      │
│  api/（控制面 REST） commands/（进程） secret/template/volume │
├─ 契约层（spec/，Copybara 同步，禁止手改）─────────────┤
│  openapi.yml（控制面） envd/*.proto（沙箱内 gRPC）      │
│  openapi-volumecontent.yml（belt） runtime-ref/belt-ref pin │
└─ 服务端（本仓库不含）─────────────────────────────────┘
   E2B Cloud（api.e2b.dev）/ BYOC / e2b-dev/runtime
```

## 3. 核心模块（文件级）

| 模块 | 路径 | 职责 | 关键符号 |
|---|---|---|---|
| Sandbox 门面 | `packages/js-sdk/src/sandbox/index.ts`（911 行） | 生命周期 API 集合 | `Sandbox.create/connect/fork/pause/kill/setTimeout/createSnapshot` |
| SandboxApi | `packages/js-sdk/src/sandbox/sandboxApi.ts`（2023 行） | 网络治理/IAM/生命周期校验 | `SandboxNetworkOpts`、`buildNetworkBody`、`createSandbox` |
| 连接配置 | `packages/js-sdk/src/connectionConfig.ts`（615 行） | 域/超时/signal/UA/merge | `ConnectionConfig`、`setupRequestController`、`wrapStreamWithConnectionCleanup` |
| envd REST | `packages/js-sdk/src/envd/api.ts` | 沙箱内 HTTP + 健康检查 | `checkSandboxHealth`、`handleEnvdApiFetchError`、`handleEnvdApiError` |
| envd gRPC | `packages/js-sdk/src/envd/rpc.ts` | 进程/文件流式 RPC | `isConnectionTerminatedMessage`、`handleRpcErrorWithHealthCheck` |
| Commands | `packages/js-sdk/src/commands/index.ts`（489 行） | 沙箱内命令执行 | `Commands.start`、`KEEPALIVE_PING_HEADER`、`supportsStdinClose` |
| 错误树 | `packages/js-sdk/src/errors.ts`（256 行） | 错误分层 | `SandboxError`、`TimeoutError`、`ServiceBusyError` |
| 重试/并发 | `packages/js-sdk/src/retry.ts`、`inflight.ts` | 429 重试 / 信号量 | `withRateLimitRetry`、`limitConcurrency` |
| IAM | `packages/js-sdk/src/sandbox/iam.ts` | token 校验 + Proxy 防护 | `iamTokenPlaceholders`、`validateIamTokenName`、`RUNTIME_PROBED_PROPS` |
| Secret/Volume/Template | `packages/js-sdk/src/secret.ts`、`volume/`、`template/` | 资源对象 | `Secret.iamToken()`、`Volume`、`dockerfileParser` |
| Code Interpreter | `packages/code-interpreter-js/src/sandbox.ts`（476 行） | runCode 高层 | `runCode`、`createCodeContext`、`JUPYTER_PORT` |
| CLI | `packages/cli/src/commands/{auth,template,sandbox}` | 命令组 | `sandboxCommand`（create/exec/fork/list/pause/resume/…） |

## 4. 生命周期（沙箱状态机）

```
              Sandbox.create(template) / fork / connect
                              │
                   ┌──────────▼──────────┐
                   │      running        │◄────────┐
                   │  (envd 0.1.0 门)    │         │ autoResume
                   └──┬──────┬──────┬────┘         │ (traffic)
              timeout│      │kill  │pause          │
         onTimeout:  │      │      │(keepMemory)   │
         pause/kill  │      │      ▼               │
                   ┌──▼──┐  ┌──▼──────────────┐    │
                   │gone │  │     paused      ├────┘
                   └─────┘  │ restore/reboot  │
                            └─────────────────┘
  证据：sandboxApi.ts SandboxState='running'|'paused'；
        onResume 'restore'|'reboot'（reboot→memory:false 冷启动）；
        keepMemory:false 禁 autoResume（filesystem-only 快照必须显式 connect）
```

## 5. 配置清单（环境变量）

| 变量 | 默认 | 证据 |
|---|---|---|
| `E2B_API_KEY` | 无 | connectionConfig.ts get apiKey |
| `E2B_DOMAIN` | `e2b.app` | connectionConfig.ts get domain |
| `E2B_API_URL` / `E2B_SANDBOX_URL` | 派生 | connectionConfig.ts |
| `E2B_DEBUG` | `false` | connectionConfig.ts get debug（true→localhost envd） |
| `E2B_USER_AGENT_SOURCE` | 无 | connectionConfig.ts（UA 溯源，regex 校验） |
| `E2B_API_CONNECTIONS` | 100 | inflight.ts（连接池） |
| `E2B_API_INFLIGHT_REQUESTS` | 1000 | inflight.ts（信号量；0=关闭） |
| CLI 凭据 | `~/.e2b/config.json` | CLAUDE.md |

## 6. 外部依赖

- 运行时库：`openapi-fetch`（REST）、`@connectrpc/connect`（gRPC-web）、`platform`（UA）、`compare-versions`（版本门）
- 上游仓库：`e2b-dev/runtime`（envd，pin `f5dc6426`）、`e2b-dev/belt`（私有 volume-content，pin belt-ref）
- 托管服务：E2B Cloud（api.e2b.dev）——SDK 是瘦客户端，逻辑主体在服务端（spec/openapi.yml 定义）

## 7. 治理与权限机制（项目内）

- `spec/README.md` + CLAUDE.md：spec 由 Copybara 同步，**禁止手改**；改规范=bump pin + `make codegen`
- `AGENTS.md`/`CLAUDE.md`：pnpm/uv、双 SDK 同步变更（JS + Python sync/async）、public surface 变更需 changeset
- `envd.yaml`：`x-internal: true` 标记控制面操作 → 沙箱代理据此生成拒绝列表（安全边界声明化）
- 令牌三级 + IAM + Secret 占位符（详见 02/03 层）
