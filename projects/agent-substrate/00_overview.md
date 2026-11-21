# 00 · Overview — Agent Substrate 考古总览（ARCH-2026-10-31-001）

## 项目一句话
**Agent Substrate**（`go.ate.dev/substrate`，Apache-2.0，2026-10-22 发布）是 Google 开源的 **Kubernetes-native 零信任 agent 运行时**：把大量"actor"（有状态 agent 类工作负载）映射到小池"worker"，空闲时快照出内存、请求到达时恢复，实现 10x 计算密度、<500ms resume、>500 activations/sec。

## 为什么值得考古（Knowledge Value）
1. **Runtime 周二号锚点**：与 10-30 已验证的 Dapr Agents（崩溃恢复/步骤持久化路线）形成**生命周期管理两条路线对照**——Substrate = F1 suspend-resume（挂起恢复、全状态快照），Dapr = F2 崩溃恢复（步骤持久化、重放不重执行）。
2. **零信任安全模型的完整工程实现**：每 actor 短时 SPIFFE X.509 + 沙箱内零凭据 + egress 网关外注凭据 + 默认拒绝网络——"凭证外注"（Strands Box→MXC→Google 跨厂商共识）的 Google 侧一手中实现。
3. **明确的"文档目标态 vs 代码实现态"分离**：`docs/concepts/architecture.md` 自标 `status: planned` + "aspirational and not yet implemented"；`docs/security/security.md` 每个原则自标 Met/In Progress/Future——为"设计意图≠实现事实"提供教科书级考古素材。
4. **OpenFGA ReBAC 授权、请求停车（request parking）、对象存储双重授权**等机制的工程细节丰富。

## 考古结论速览（详见各层）
- **Project Layer**：14 个二进制（ateapi/atelet/ateworker-gvisor/ateworker-microvm/atenet/kubectl-ate…）+ 52 个 internal 包；Go 1.27 / 1466 非 vendor .go 文件 / 43.9 万行 / 689 测试文件。
- **Engineering Knowledge**：40 条 EK（EK Graph，6 类边连接，含机制/因果/依赖/约束/对照）。
- **Knowledge Layer**：7 个 Core KO（L3-L5，全部带 aggregation_rule R1-R4）。
- **Flow Atlas**：七类流（Control/State/Data/Evidence/Authority/Memory/Policy）均可回溯到具体文件/符号。
- **Candidates**：5 个未验证假说（含跨项目对照）。
- **Validation**：Truth/Coverage/Flow/Abstraction/Counterexample/Epistemic 六审计 + 盲重建独立判定（Independent Validation Report 见 06_validation/）。

## 关键认知增量（相比已知文献）
1. **请求停车（request parking）不是 commit 语义**：预算（默认 5s）只限新重试启动，已 in-flight 的 resume 永不取消——"晚到服务"（serve late）而非失败。这是超卖系统对瞬时饱和的务实处理，区别于 Dapr 的重试即重放。
2. **快照完成标记（checkpoint-done.json）+ BootID 校验**：worker 崩溃/超时后，atelet 在同一 boot 上可复用已完成的 checkpoint（幂等恢复），跨 boot 必须重做——"幂等以 boot 为界"。
3. **egress 凭据注入错误映射**：NotFound/PermissionDenied → 403（重试无意义）；Unavailable/DeadlineExceeded → 503（fail-closed 可重试）；injectionTimeout=4s（小于 ext_proc message_timeout，慢 provider 变 503 而非 Envoy 504）。
4. **OpenFGA bootstrapOwners 用 contextual tuple**（每次 Check 附带、非存储 tuple）——全局 owner 不可经 API 撤销、随配置失效。设计上规避"API 可撤销的超级管理员"。
5. **AGENTS.md 是最强工程治理文档之一**：禁 AI 味词汇清单、agent 不得 sign-off DCO、atelet/worker 互不透明（仅共享 ateworkerpb+nodepath）、PR 单改动原则——"AI 协作规则显式固化"的行业标杆。

## 质量指标（详见 run_metadata.yaml）
- 层配比：Project Layer 1 / EK 40 / KO 7 / Candidates 5
- EK Graph：平均出边 ≥1、游离 EK 0%、聚合规则覆盖率 100%
- Validation：IVR 判定统计（CONFIRMED/PARTIALLY_CONFIRMED/DOWNGRADED 等）见独立报告
- 反例攻击：每个 L3+ KO ≥3 定向反例

## 证据基线
- 仓库：https://github.com/agent-substrate/substrate.git @ `542eca1e348e80001dcca48789e9433a84eeb214`（main，2026-10-30）
- Skill 版本：knowledge-archaeology v3.2
- 考古方式：静态代码考古（Go/K8s 代码 + docs + e2e 测试），无 Docker 环境不实跑集群
