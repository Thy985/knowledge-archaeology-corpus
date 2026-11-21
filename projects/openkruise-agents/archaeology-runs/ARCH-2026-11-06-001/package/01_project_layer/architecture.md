# 01 Project Layer — Architecture

## 1. 三层部署形态

```
                         ┌──────────────────────────────────────────┐
                         │  Control Plane: agent-sandbox-controller │
                         │  7 controllers (controller-runtime)      │
                         │  CRD: Sandbox / SandboxSet / Claim /     │
                         │       UpdateOps / Checkpoint / Commit /  │
                         │       PoolAutoscaler / TrafficPolicy ... │
                         └───────────────┬──────────────────────────┘
                                         │ watches / patches CRs
              ┌──────────────────────────┼─────────────────────────────┐
              ▼                          ▼                             ▼
   ┌────────────────────┐     ┌──────────────────────┐     ┌────────────────────┐
   │ sandbox-manager    │     │ sandbox-gateway      │     │ agent-runtime      │
   │ API(E2B)+Manager   │     │ JWT/mTLS + route     │     │ in-sandbox sidecar │
   │ +Infra(sandboxcr)  │◄───►│ + wake-on-traffic    │     │ (E2B envd compat)  │
   │ +quota(anti-drift) │     │ + peer gossip        │     └────────────────────┘
   └────────────────────┘     └──────────────────────┘
```

来源：`cmd/` 8 个入口；`pkg/controller/controllers.go:SetupWithManager`；`AGENTS.md` Control Plane Layering。

## 2. 分层与依赖规则（治理事实）

`AGENTS.md:14-100` 明确三层与依赖方向：

| 层 | 路径 | 职责 | 禁止 |
|---|---|---|---|
| API | `pkg/servers/**` | E2B 兼容协议、路由、HTTP 语义、认证、状态码映射 | 不得把协议模型/兼容规则下推到 Manager/Infra |
| Manager | `pkg/sandbox-manager/**`（除 infra） | 协议无关业务规则：生命周期、配额、准入、释放策略 | 不得依赖 `pkg/servers`；不得直接读写后端状态资源 |
| Infra | `pkg/sandbox-manager/infra/**` | K8s 后端能力（sandboxcr 实现） | 不得依赖 API 模型/HTTP/认证语义/Manager 策略 |
| Controller | `pkg/controller/**` + `cmd/agent-sandbox-controller` | 独立 operator | **依赖闭包不得包含 `pkg/servers/**` 或 `pkg/sandbox-manager/**`** |

其他硬规则（`AGENTS.md`）：
- `pkg/features` 仅 controller 可用，manager/servers 不得导入。
- 控制器不得复用 sandbox-manager 的 API 行为/业务编排/Infra 实现（`AGENTS.md:61-66`）。
- **sandbox-manager 部署中 recycle 视为不存在**；独立 controller 保留 recycle 能力，但不代表 CR 可跨交付复用（`AGENTS.md:14-26`）。
- 新增 API/架构变更需 `docs/proposals/`（仓库当前无 docs 目录 → 提案机制在代码库中不可见，属文档缺口）。
- 生成文件（`client/`、`proto/`、`config/crd/`）禁止手改。

## 3. 模块规模（非测试 Go 行数，来源 `wc -l`）

| 模块 | 行数 | 角色 |
|---|---|---|
| pkg/controller | 74,203 | 控制面状态机（sandbox 1,261 + metrics 1,016 + core/* 4,000+；sandboxupdateops 656；poolautoscaler 644+545；sandboxclaim 580；sandboxset 565+272；commit 293+316；securitytokenrefresh 418） |
| pkg/sandbox-manager | 38,333 | 数据面：infra/sandboxcr claim 1,406 / clone 789 / sandbox 728；quota antidrift 630 / redis 400；core 512；api 465 |
| pkg/servers | 31,332 | E2B 兼容 API |
| pkg/utils | 28,649 | 共享工具（**AGENTS.md 明确不得新增 catch-all utils**——存量债务） |
| pkg/identity | 9,122 | 身份/令牌 |
| pkg/cache | 8,075 | informer/cache provider |
| pkg/sandbox-gateway | 7,890 | filter 518 / server 418 / jwtauth 348 / config 345 / runtimecredentials 262 / wake 206 / controller 235 |
| pkg/webhook | 7,411 | 准入/默认值（pod/poolautoscaler/sandboxset/sandboxtemplate/sandboxupdateops/securityprofile） |
| pkg/agent-runtime | 5,247 | 沙箱内 sidecar |
| pkg/tracing | 4,185 | Reconcile/controller span + trace-log 关联 |
| pkg/agentidentity | 4,066 | 身份提供者 |
| pkg/cli | 3,737 | okactl |

## 4. 八个二进制与职责

| 二进制 | 入口职责 | 证据 |
|---|---|---|
| `agent-sandbox-controller` | 7 控制器注册 | `controllers.go:30-66` |
| `sandbox-manager` | 数据面 API/配额/路由/worker id | `pkg/sandbox-manager/{api.go,core.go,routes.go,worker_id_allocator.go}` |
| `sandbox-gateway` | 流量入口、JWT、route registry、wake | `pkg/sandbox-gateway/{server.go,filter/*,jwtauth/*,wake/*}` |
| `sandbox-gateway-cert-init` | 证书初始化 | `cmd/sandbox-gateway-cert-init` |
| `agent-runtime` | 沙箱内 runtime（E2B envd 兼容） | `pkg/agent-runtime` |
| `agent-identity-provider` | 令牌签发 | `pkg/agentidentity` |
| `commit-job` | Commit CRD 的 commit/push（nerdctl） | `pkg/controller/commit` |
| `okactl` | CLI | `pkg/cli` |

## 5. 外部依赖（go.mod）

- `k8s.io/* v0.37.2`（api/apimachinery/apiserver/client-go/apiextensions-apiserver）
- `sigs.k8s.io/controller-runtime v0.25.2`
- `connectrpc.com/connect v1.19.1`（gateway/manager 协议）
- `github.com/hashicorp/memberlist v0.5.4`（peer gossip）
- `github.com/envoyproxy/go-control-plane`（L7 网关）
- `go 1.26.0`；kubebuilder v4 layout，group `agents`，domain `kruise.io`

## 6. 关键架构张力（Observation，非缺陷断言）

1. **单一 operator 承载 7 类控制器**：Sandbox/SandboxSet/Claim/UpdateOps/Commit/PoolAutoscaler/securitytokenrefresh 全部在 `controllers.go` 内注册，共享 `Deps{MetricsCleanup, RuntimeTLSBundle}`。耦合点集中在 `Deps`，扩展靠追加字段而非改 Add 签名（`controllers.go:22-29` 注释明确该设计）。
2. **控制面/数据面双实现**：controller 可独立 reconcile CRD，sandbox-manager 另有 infra/sandboxcr 实现；AGENTS.md 用"禁止依赖"隔离，但两者对 Sandbox 语义的理解必须靠契约维持（如 recycle 在 manager 侧不存在）。
3. **生成代码占比高**：71 个生成文件（deepcopy/client/listers），接口事实一致但知识价值低。
