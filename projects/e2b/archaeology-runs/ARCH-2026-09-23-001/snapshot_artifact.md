# Repository Snapshot Artifact — e2b-dev/E2B

> 阶段 3 产物。本文件只记录可追溯到仓库实际内容的事实，不做 Knowledge Synthesis。
> 来源：浅克隆 `/home/user/.doubao/agent_mode/workspace/archaeology-jobs/ARCH-2026-09-23-001/repo`（HEAD `ccaf9fc`）。

## Snapshot 记录

| 字段 | 值 |
|---|---|
| repository | https://github.com/e2b-dev/E2B.git |
| commit SHA | `ccaf9fc0ffe6ac39c7ec786af7608ab1de19467b` |
| branch / tag | `main`（浅克隆默认分支；commit subject: "[skip ci] Release new versions", 2026-09-18 12:06:48 +0000） |
| analysis timestamp | 2026-09-23（SOP 日度轮次） |
| repository version | 根包 `e2b`；js-sdk `2.51.0`；python-sdk `2.51.0`（packageManager pnpm@10.34.5） |
| current knowledge-archaeology-skill version | 3.2（生产版，本地 `.user_skills/knowledge-archaeology/`） |
| GitHub 元数据（阶段 2 探测） | 13,920★；pushed_at 2026-09-22 |

## 项目基础地图

```
E2B/
├── packages/                        # pnpm monorepo（双 SDK + CLI）
│   ├── js-sdk/                      # 主 TypeScript SDK（src 51 个 .ts 文件，发布名 e2b）
│   │   ├── src/
│   │   │   ├── client.ts            # E2B client：绑定 connectionOpts 到子类静态 boundOpts
│   │   │   ├── connectionConfig.ts  # ConnectionConfig：域/超时/signal/代理/keepalive/mergeOpts
│   │   │   ├── sandbox/
│   │   │   │   ├── index.ts         # Sandbox 生命周期门面（create/connect/fork/pause/kill/…）
│   │   │   │   ├── sandboxApi.ts    # SandboxApi：网络治理/IAM/生命周期校验（2023 行）
│   │   │   │   ├── network.ts       # 仅导出 ALL_TRAFFIC = '0.0.0.0/0'
│   │   │   │   ├── iam.ts           # token 名校验 + Proxy 陷阱防护
│   │   │   │   ├── signature.ts     # 签名 URL（sha256(path:operation:user:token) → v1_ base64）
│   │   │   │   ├── mcp.ts           # MCP gateway 集成
│   │   │   │   └── rpc.ts           # envd gRPC 错误映射 + health-probe 归因
│   │   │   ├── envd/
│   │   │   │   ├── api.ts           # envd REST 客户端 + 健康检查 + 错误映射
│   │   │   │   ├── rpc.ts           # connectrpc 客户端（CONNECTION_TERMINATED_MESSAGES）
│   │   │   │   └── filesystem/      # 文件系统 gRPC
│   │   │   ├── commands/index.ts    # Commands：/bin/bash -l -c 执行 + keepalive + stdin close
│   │   │   ├── api/                 # 控制面 REST 客户端（http2 fetch 工厂/retry/inflight）
│   │   │   ├── errors.ts            # 错误树（SandboxError 基类 + statusCode）
│   │   │   ├── retry.ts             # 429+Retry-After 重试
│   │   │   ├── inflight.ts          # FIFO 信号量 limitConcurrency
│   │   │   ├── http2.ts             # fetch 工厂 per-proxy 缓存
│   │   │   ├── secret.ts            # Secret（write-only，${e2b.secrets.<name>}）
│   │   │   ├── volume/              # Volume（client/index/types/schema.gen）
│   │   │   └── template/            # 模板构建（dockerfileParser/readycmd/buildApi）
│   │   └── tests/                   # api/bundle/client/envd/sandbox/secret/template/volume + runtimes
│   ├── python-sdk/                  # 同步（sandbox_sync）+ 异步（sandbox_async）双实现
│   │   └── e2b/
│   ├── code-interpreter-js/         # runCode 高级层（Jupyter 直连 /execute，context 保留状态）
│   │   └── src/                     # sandbox.ts 476 行 + messaging.ts + charts.ts + consts.ts
│   ├── code-interpreter-python/     # Python 对应实现
│   ├── desktop-js/ + desktop-python/ # 桌面运行时代理
│   └── cli/                         # Commander CLI（auth/template/sandbox 三大命令组）
├── spec/                            # Copybara 同步的 API 规范（禁止手改）
│   ├── openapi.yml                  # 控制面 API（E2B Cloud 服务契约）
│   ├── openapi-volumecontent.yml    # belt 私有仓库规范（belt-ref pin）
│   ├── envd/                        # envd.yaml + filesystem/process proto（runtime-ref pin）
│   ├── mcp-server.json
│   └── runtime-ref / belt-ref       # 上游 pin commit
├── scripts/                         # fetch-spec.sh / codegen 等
├── templates/                       # base / httpbin 等沙箱模板
├── copy.bara.sky                    # Copybara 配置
├── AGENTS.md                        # 仓库内 Agent 约束（pnpm/uv、双 SDK 同步、changeset）
├── CLAUDE.md                        # 维护指南（同上约束 + TASTE.md 设计原则）
├── TASTE.md                         # SDK 设计原则
└── redocly.yaml                     # SDK 生成前 openapi tag 过滤
```

## 识别清单

| 维度 | 识别结果 |
|---|---|
| **主要语言** | TypeScript（js-sdk/cli/code-interpreter-js）+ Python（python-sdk，sync/async 双实现）；proto（envd 契约） |
| **主要运行入口** | SDK 为库，无服务器运行时。用户入口：`Sandbox.create()/connect()`（js-sdk 与 python-sdk）；CLI 入口 `packages/cli/src/index.ts`；`code-interpreter` 的 `Sandbox.runCode()` |
| **核心模块** | `sandbox/`（生命周期+网络+IAM）、`envd/`（沙箱内运行时通道）、`api/`（控制面 REST）、`commands/`（进程执行）、`secret/template/volume`（资源对象）、`cli` |
| **核心数据结构** | `SandboxOpts/SandboxInfo`、`SandboxNetworkOpts/Info/Update`、`SandboxIamOpts`、`SandboxLifecycle`（onTimeout 判别联合）、`SandboxState ('running'|'paused')`、`McpServer`、`ProcessConfig`（proto）、`StartResponse/ConnectResponse`（proto） |
| **核心状态** | 沙箱状态机 running→paused（pause/kill/autoResume/onTimeout）；sandbox domain 派生 `${port}-<sandboxId>.e2b.app`；envdVersion 版本门（create 后 <0.1.0 → kill+TemplateError） |
| **主要测试体系** | 169 个 `*.test.ts`（vitest）；live-sandbox 集成测试（testTimeout 60s、E2B_TEST_MAX_WORKERS）；js-sdk tests/{api,sandbox,secret,template,volume,runtimes}；code-interpreter-js tests/{killedSandbox,statefulness,systemd,reconnect,contexts,…}；cli tests/commands/sandbox/* |
| **主要配置** | 环境变量：`E2B_API_KEY`、`E2B_DOMAIN`（默认 e2b.app）、`E2B_API_URL`、`E2B_SANDBOX_URL`、`E2B_DEBUG`、`E2B_USER_AGENT_SOURCE`、`E2B_API_CONNECTIONS`（默认100）、`E2B_API_INFLIGHT_REQUESTS`（默认1000）；`~/.e2b/config.json`（CLI 凭据）；`packages/*/package.json`（changeset 版本） |
| **权限 / policy / governance 机制** | ① token 三级：API key（控制面）→ envdAccessToken（X-Access-Token）→ trafficAccessToken（E2B-Traffic-Access-Token）/ MCP GATEWAY_ACCESS_TOKEN；② IAM workload identity（`${e2b.identity.tokens.<name>}` placeholder，egress proxy 按请求签发）；③ Secret write-only（`${e2b.secrets.<name>}`）；④ x-internal 控制面隔离（envd.yaml 标记 → 沙箱代理生成拒绝列表）；⑤ spec/ 所有权边界（Copybara pin，禁止手改）；⑥ AGENTS.md/CLAUDE.md 双 SDK 同步 + changeset 治理 |
| **主要外部依赖** | openapi-fetch、@connectrpc/connect（gRPC-web）、platform（UA 探测）；上游 e2b-dev/runtime（envd，pin `f5dc6426`）、e2b-dev/belt（volume-content，私有）；E2B Cloud 托管服务（api.e2b.dev）；pnpm/uv 工具链 |

## 证据可追溯性说明

- 所有路径均为浅克隆 `repo/` 内实际路径；commit/版本号来自 `git rev-parse HEAD` 与 `package.json`/`pyproject`（读取时点）。
- `spec/` 规范由 Copybara 同步（spec/README.md），本仓库非规范源头；`runtime-ref` 内容为 pin commit `f5dc6426ef45b6b286ff60c11fc6bba7e00db95e`。
- 本 Snapshot 不包含对 E2B 托管服务的运行时观测（仅代码/文档/测试证据）。
