# 05 · Candidates — 未确认内容（Hypothesis / 跨项目假说 / 待验证模式）

> 与 KO 严格区分：以下全部是未验证假说，不是知识。每条标注缺失证据与验证路径。

## C-1 跨厂商"凭据在盒子外"共识假说
- **假说**：Agent Substrate 的"沙箱内零凭据 + egress 网关外注"与 Strands Box→MXC→Google 的跨厂商凭据外注共识是同一安全原则的多实现。
- **当前证据**：Substrate 侧 S3（security.md Zero Ambient Credentials Met + credentials.go 外注实现）；他方仅知背景。
- **缺失证据**：其他实现（MXC/Strands）的对照代码。
- **验证路径**：考古 Strands/MXC 仓库确认同一原则。
- **状态**：Hypothesis（跨项目对照未验证）。

## C-2 "幂等以 boot 为界"是否普遍适用
- **假说**：快照/缓存复用若以 bootID 为有效期边界，可推广为通用模式（任何"节点重启后文件可能 torn"的场景）。
- **当前证据**：Substrate S3（checkpoint.go BootID 校验）。
- **缺失证据**：其他项目复用。
- **验证路径**：在文件缓存/镜像缓存类项目中找对照。
- **状态**：Hypothesis。

## C-3 memborrow（跨 worker 内存借用）实现细节未确认
- **观察**：internal/ateworker/memborrow/ 目录存在，但本次考古未逐行确认其机制（借用方向/归还策略/限额语义）。
- **当前证据**：S2（目录存在）。
- **缺失证据**：机制细节、与 cgroup/memwatch 的交互。
- **验证路径**：后续考古深读 memborrow/ 代码。
- **状态**：Observation（非知识，不写入 KO）。

## C-4 文档状态标注新鲜度未验证
- **观察**：security.md 标注 In Progress 的条目（如 resetActorDirs、OpenFGA 网络策略）当前真实实现程度未逐一核对；README 的北极星性能数字是 target 还是实测未在本考古范围确认。
- **当前证据**：docs 标注本身（S2）。
- **缺失证据**：与当前代码的逐条 diff。
- **验证路径**：按 security.md 每个 In Progress/Future 条目回查代码。
- **状态**：Observation。

## C-5 benchmarking 是否已实证北极星指标
- **观察**：internal/benchmarking（62 文件）承载指标验证，但本次静态考古无法确认集群级实测（20 万 agent/500 activations/sec）结果。
- **当前证据**：S2（目录 + README 目标数字）。
- **缺失证据**：CI/发布中的实测报告。
- **验证路径**：查 CI artifacts / releases 中的 benchmark 输出。
- **状态**：Hypothesis（未验证为事实）。

---

## 候选→知识升级门槛
以上任何候选升级为 KO 需：①跨项目证据（S8）或 ②用户/维护者确认（S7）+ 完整证据链。当前全部保持 Hypothesis/Observation。
