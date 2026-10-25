# Agent Substrate 知识考古总览（00 Overview）

> run_id: ARCH-2026-10-26-001 · commit `59a58a5f`（v0.6.0 后）· skill knowledge-archaeology v3.2 · 仓库 https://github.com/agent-substrate/substrate.git

## 一句话定位
**零信任 agent 沙箱运行时**（Go/K8s-native）：把海量"actor"（有状态 agent 工作负载）映射到少量热"worker"上，actor 空闲时整机快照出内存、请求到来时恢复，换取密度与亚秒级激活；每个 actor 有身份、无凭据、无开放网络、无法逃出沙箱（"Every agent has an identity, but no credentials, no open network, and no way out of the sandbox"——docs/security.md）。

## 六条认知主线

1. **生命周期与恢复三件套（快照-恢复范式）**：分级快照（VOLUMES/ROOTFS/MEMORY）+ 幂等 checkpoint（`.checkpoint-done.json` 完成标记，超时/重启重试直接复用）+ 本地/远端双持久化（locality + durableSnapshot PENDING/COMPLETED）。与 dapr-agents（10-25 已考古）的 durable execution 三件套（每步持久化 + 幂等活动 + 确定性 ID）构成同一问题的两种解法：**整机快照 vs 步骤日志**。
2. **零信任四支柱**：必沙箱（gVisor/CH microVM，明文容器不支持）+ 零环境凭据（凭据只在水合时出现）+ 默认拒绝网络（egress fail-closed PEP + ingress mTLS allowlist）+ 组件间 mTLS 全覆盖（1h SPIFFE X.509）。
3. **身份与代行可区分**：actoridjwt 双 Kind（KindActor `actor/...` vs KindWorkerForActor `worker-for-actor/...`）——actor 自己的代码与 Substrate 组件代行同一 actor 时主体不同，依赖方可授予不同权限；ClaimsToWire 拒绝未设 Kind 的 Claims（防漏签发）。
4. **请求停车而非立即失败**：worker 池饱和时 router 挂起请求（parking lot 容量 1024 + budget 5s + 指数退避重试），ResourceExhausted/FailedPrecondition/Unavailable 视为可重试——把瞬时资源压力从客户端 503 转成内部重试。
5. **低意见运行时（not an SDK）**：运行标准 OCI 镜像，安全强制在 workload 外；atelet 与 worker 运行时互不透明（只共享 ateworkerpb + nodepath）——框架无关，agent 无需改 SDK。
6. **治理代码化 + 愿望/实现显式分离**：AGENTS.md 强制 AI 贡献披露、fail loudly、pin image digest、CI 只读 fail-closed 谓词；docs/architecture.md 明确声明"Much of this architecture is aspirational, and is not yet implemented!"，security.md 用 Met/In Progress/Future/Out of Scope 标注每个原则——**设计意图≠实现事实被制度化**（考古的审计锚）。

## 关键数字（README 声明，可复现路径 benchmarking/ 存在，本轮未实测）
- <400ms resume · 1500+ activations/sec（README；docs/architecture.md North Star：100ms p95 / 10 亿 actors / 1000 wakeups/sec——目标非现状）
- v0.6.0（2026-10-22 release）· 5785★/627 forks（GitHub API 2026-10-26）
- 1425 Go 文件 ≈ 42.7 万行 · 670 测试文件 · go 1.27

## 与 Corpus 已有项目对照（跨项目连接）
- dapr-agents（10-25）：durable execution（步骤持久化+重放检测） vs Substrate（整机快照+恢复）——"恢复"两种哲学；HITL/审批 vs 零信任网络策略——遏制机制对照
- MXC / Strands Box（Harness 周 validated）：OS 级隔离 + 凭据外挂 vs Substrate 零信任 + egress 凭据注入后丢弃——**凭据外注证据线三仓收敛（Google 官方实现）**
- DeepSeek Harness（validated）：sandbox/approval/credentials seam vs Substrate 硬隔离默认拒绝——从 harness 到 runtime 的强化对照
