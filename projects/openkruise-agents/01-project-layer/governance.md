# 01 Project Layer — Governance, Authority & Delivery Boundaries

## 1. 架构治理是"可执行约束"，不只是文档

`AGENTS.md` 在仓库根目录，属于 agent/编码规范；其中分层规则与依赖方向具备**强制语义**（"normative for new work"、"Existing cross-layer imports are legacy debt, not architectural precedent"）：

1. **API → Manager → Infra 单向依赖**；禁止 API→Infra 捷径、禁止下层依赖上层。
2. **controller 依赖闭包**不得包含 `pkg/servers/**` 或 `pkg/sandbox-manager/**`；共享代码必须移到真正中立的包，工具包不得隐藏间接反向依赖。
3. `pkg/features` 仅 controller 可用；manager/servers 若需可选行为，优先"单次调用可控制的接口/参数"，避免进程级开关。
4. **Kubernetes 读取纪律**：禁止用 `APIReader` 做 List；`Get` 仅在 informer 不能满足正确性/功能且说明理由并获批准后使用。
5. `cmd/*` 只装配依赖与启动组件，业务逻辑归各层。
6. 不新增 catch-all `utils/common/helpers` 包；新抽象必须有具体当前需求。
7. 生成文件（client/proto/config/crd）禁止手改；新增 API/架构变更需 `docs/proposals/`。
8. 提交必须 sign-off（`git commit -s`）。

**知识含义**：这套规则的目的是让"控制面 operator"能独立于"数据面服务"演进——controller 不因 manager 的业务模型变化而被迫耦合，反之亦然。

## 2. 部署语义边界（recycle）

- **sandbox-manager 部署**：recycle 视为不存在；manager 的 API/Manager/Infra/cache/quota/token 均不得添加 recycle 特殊处理或兼容。
- **独立 controller**：保留 recycle 能力（`EnsureSandboxRecycled`），但"能 recycle"不等于"CR 可跨部署复用"。
- 代码佐证：`core/recycle.go` 存在且 controller 分派 Recycling；`sandbox-manager` 无 recycle 路由。

## 3. 安全边界

### 3.1 网关鉴权
- `pkg/sandbox-gateway/jwtauth`：JWT 验证器由 OIDC 配置异步加载；状态机 `Disabled / AwaitingConfig / Initializing / Ready`；重复 `Configure` 必须与首次完全一致（`validateRepeatedConfiguration`），否则报错。
- `runtime-mtls` 配置目录存在（gateway 与 runtime 间 mTLS）。
- `runtimecredentials`（262 行）负责 runtime 侧凭据。
- 安全策略（`SECURITY.md`）：漏洞不建 Issue，走 `kubernetes-security@service.aliyun.com`；advisory 在修复版本发布后公开。

### 3.2 身份提供者
- `SecurityIdentityProviderGate` 默认 false：关闭时用随机 UUID 生成访问令牌；开启时走外部 identity provider 签发（`pkg/agentidentity`）。
- `security/v1alpha1` 提供 AgentIdentity/AgentRole/AgentRoleBinding/CredentialProvider —— 身份与授权模型是 CRD 化、可审计的。

### 3.3 沙箱内执行边界
- `agent-runtime` 作为 sidecar 运行（E2B envd 兼容），控制面通过 runtime 注入/初始化；`RuntimeInitialized` 条件在每次 resume 重置 → 确保恢复后 runtime 重新初始化与 CSI re-mount。
- `PodTemplateDelta` 的存在意味着恢复不是"信任模板"，而是"在模板基础上应用运行时补丁"，可减少恢复偏差。

## 4. 交付边界（本次考古）

- 目标仓库**只读**：未修改任何文件。
- KnowlegeMap / knowledge-archaeology-skill 只读；生产 skill v3.2 未改动。
- 结果写入 `knowledge-archaeology-corpus`，每项目顶级目录，不覆盖历史。
- 飞书落盘**未授权**，未执行。
- 无系统性 skill 缺陷证据 → 本轮不发起 Skill Evolution（见 06 与收尾说明）。

## 5. 治理观察（Observation）

1. `AGENTS.md` 同时是"给 AI coding agent 的指令"和"人类架构规范"，两者共用一套文本；好处是单源，风险是规则随 agent 工具演进而漂移。
2. 规则中"禁止 APIReader List / Get 需批准"是**可检查的工程纪律**，但仓库内没有自动化 lint 强制（未发现对应 CI 规则）；依赖 review。
3. `docs/proposals/` 被要求但仓库无 docs 目录（`ls` 仅根级 md）→ 提案流程对考古者不可见，属治理透明度缺口（C-05）。
4. AGENTS.md 明确"不要新增 Prometheus 指标除非用户显式要求"——与可观测性目录（prometheus/otel）并存，说明指标增长受控。
