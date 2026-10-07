# PullFrog Candidates（未验证假设 / 跨项目假说 / 暂定模式）

> 全部标 Hypothesis，不进知识层。区分"项目内观察"与"跨项目假说"；每条给当前证据 / 缺失证据 / 验证路径。

---

**C-01 · 单 session 复用的冷启动经济学（跨项目假说）**
opencode harness 的 post-run gate retry 与 reflection 复用同一 session（warm MCP/plugins/provider），避免冷启动成本。假说：session 复用对长尾小任务（<2min）收益显著，对长任务（>10min）收益趋近于零，且 session 内状态污染（长上下文累积）会抵消收益。
证据：`agents/opencode.ts`（"same session — warm MCP/plugins/provider/context, avoids cold-start"）。缺失：无跨任务基准数据。验证路径：对同一 repo 连续 run 测 token 消耗与延迟分布。

**C-02 · stop hook 禁用后的信任落差（项目内未决）**
2026-05 生产审计 8/9 配置脚本是 foot-gun → stopHook 在 collectPostRunIssues 中禁用（#714）。未决：用户在 PullFrog console 配置了 stop script，期望它把关，但运行时已不执行——信任落差是显式的，等待产品决策（重启用条件/或 UI 标记 deprecated）。
证据：`agents/postRun.ts` 禁用注释。缺失：服务端 UI 文案状态（不可见）。验证路径：查服务端 repo / 控制台。

**C-03 · in-process harness vs 子进程 NDJSON 的通用性（跨项目假说）**
PullFrog 从 CLI-subprocess NDJSON + `--continue` respawn 迁移到 `opencode serve` in-process harness（事件流 + session.prompt 复用），收益是"不重新 spawn 也能延续上下文"。假说：对任何提供 serve/session API 的 agent 后端，in-process harness 优于子进程 NDJSON，除非后端 SDK 不稳定。
证据：`agents/opencode.ts` 头部注释。缺失：Claude/Codex 侧仍用子进程——未做同构迁移。验证路径：对比三 harness 的失败率与 token 效率。

**C-04 · "等待优于 abort"的 MCP timeout 权衡（跨项目假说）**
opencode MCP timeout 660s（> checkout_pr 600s）是"宁可等答案也不 abort 重试"的显式选择（#860/#864 git lock 破坏性修复；300s 时 #1171 一半概率 abort 客户端随服务端继续）。假说：当工具调用有副作用且 abort 后重试成本高（git lock 损坏）时，长 timeout + 信任服务端最终返回 优于 短 timeout + 重试。
证据：`agents/opencode.ts` MCP timeout 注释。缺失：无对照数据（short-timeout 对照组因事故被移除）。验证路径：构造可重入工具 vs 不可重入工具的对比实验。

**C-05 · passOnTimeout 测试语义的可推广性（跨项目假说）**
tokenExfil 测试 `passOnTimeout: true`——持久 agent 跑满预算证明沙箱守住（而非攻破）。假说：对"开放性安全探测"类测试（agent 可无限尝试），超时即通过是合理语义；对"确定性功能"测试则不可用。判定标准：测试意图是"证明不可达"而非"证明可达"。
证据：`test/crossagent/tokenExfil.ts` 注释（"a persistent agent can still be trying when the budget runs out — that proves the sandbox held, not that it broke"）。缺失：无同类测试范式对照。验证路径：对比其他安全基准（如 CyberSecEval）的超时处理。

**C-06 · Router 分层（jev|scorer|heuristic|fixed）与 tier/stakes 的架构（服务端不可验证，推断）**
runType/routing.tier 由服务端下发，PayloadRouting 判别联合含 jev（Jev 打分器）/scorer/heuristic/fixed。假说：PullFrog 的路由决策是"价值密度"路由（低成本模型处理低 stakes 事件、Jev 打分用于 PR 排序），与 OSS/managed 的 proxy mint 联动。证据：`external.ts` PayloadRouting + `utils/runContext.ts`。缺失：路由算法本体在服务端。验证路径：收集真实 run 的 routing.tier 分布 + 模型选择相关性。

**C-07 · learnings 字节对比与 xrepo 双轨（跨项目假说）**
repo learnings（run 内 agent 维护）与 xrepo learnings（org 级 agent-curated）双轨并存，且 seed 时字节对比防重复注入。假说：记忆系统"按 repo 隔离 + org 级汇总"优于单一全局记忆，因 repo 间上下文污染小于 org 级抽象收益——但 xrepo 未做 token 预算验证。证据：`utils/runContext.ts` xrepoBrief/xrepoLearnings + `main.ts` learnings seed。缺失：xrepo run 的 token 增量数据。验证路径：对比 xrepo on/off 的 token 消耗与 review 质量。

**C-08 · Intelligence ≠ Authority 的同构验证（跨项目假说）**
本考古的 KO-08（执行面与决策面解耦）与用户在 Tafcm 验证过的 "Intelligence ≠ Authority"（判断力与执行权是两个独立维度）结构同构。假说：这是 agent 系统的稳定认知模型——PullFrog（token 分层/subagent 门控/交付门控）与 Tafcm（受控接口/证明与执行分离）是两个独立证据点。
证据：KO-08 + 本仓库三处实例。缺失：第三个独立项目证据。验证路径：考古第三个含权限模型的 agent 项目（如 openclaw/rampart 已有 corpus 可对比）。

---

## 暂定模式（Pending Pattern，未达 L3 标准）

**PP-01 · "debug 自喂活性"陷阱**：`utils/activity.ts` 把自己的 spawn/process activity 调试行显式过滤，防 debug 模式自喂 watchdog 活性。观察：任何活性检测系统若把"自己的日志"计入活性，debug 开启时 watchdog 永久失效。单案例，未验证为模式。
**PP-02 · "重试同 token 无效，re-mint 才是解药"**：GitHub 偶发 mint 出 git edge 永不接受的 token（#1115），retry 同 token 永远 401。观察：token 类故障的"重试"对象应是 mint 动作而非请求。与一般 5xx 重试策略形成对照（哪些故障类需要"更换身份"而非"重放请求"）。
