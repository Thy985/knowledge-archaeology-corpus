# PullFrog Knowledge Layer（Generalized KO）

> 层：L3 Pattern / L4 Cognitive Model / L5 Methodology（窄尖顶）。8 个 KO，全部带 aggregation_rule（R1 机制簇 / R2 因果链簇 / R3 不变量簇 / R4 主题簇）+ 簇内 EK + 边类型。Epistemic 状态严格标注：未跨项目验证的模式标 `Cross-project validation pending`。
> 纵向链要求：每个 KO 可回溯到 02 EK Graph 的具体簇；EK 不因形成 KO 删除。

---

## KO-01 · Fail-Closed 拒绝启动：配置不完整或门控失效时，宁可让 run 失败也不带病运行（L4 Cognitive Model）

**aggregation_rule**: R3 不变量簇（fail-closed 不变量）
- 簇内 EK: EK-16（CI && 沙箱 none → throw）、EK-23（subagent 禁调用集派生空集 → throw）、EK-05（trial fallback 后必须 re-resolve agent）、EK-11（MCP token 刷新失败不降级 scope）、EK-06（agent key 启动前 fail-fast）
- 边类型: mechanism（共享"配置失效拒绝启动"）+ constraint（EK-16/EK-23 是硬门，EK-05/EK-11 是软门）
- 簇成立: 内聚（同一不变量"不降级启动"）；跨实例（沙箱探测、工具派生、凭证刷新、模型解析 4 个独立子系统）；解释范围扩大（从单事故到通用原则）；可命名（Fail-Closed 拒绝启动）；可回溯（每条 EK 带证据）

**一句话**：在不可信代码上执行可信 agent 时，安全机制的任何一部分失效都必须转化为显式失败，而不是静默降级——因为"带病运行"的失败模式不可预测（可能送错模型、可能暴露 secrets、可能跨 PR clobber）。

**L1→L4 链**（每层可回溯）：
- L1: CI 下 shell 沙箱三级探测全失败 → `spawnShell` throw 并给诊断（EK-16）；subagent 禁调用集派生空 → throw（EK-23）
- L2: 因为静默降级会让 agent 获得无沙箱 shell / 无门控工具面，而 run 仍被用户视为"受控执行"（EK-16 注释"绝不能静默降级到无沙箱"）
- L3: 同类模式——TLS 证书校验失败拒绝连接、CI 缺 secret 直接红（fail-closed 是安全系统的通用解）
- L4: **Fail-closed 拒绝启动**——安全不变量失效与功能故障必须用不同信号表达（前者必须终止）

**Epistemic**: Validated Pattern（项目内强证据：4 个独立子系统 + 对抗测试断言）；跨项目验证 pending。

---

## KO-02 · 事故驱动防御链：一次真实事故（subagent 越过权限改共享工作树）催生三层防御（L3 Pattern）

**aggregation_rule**: R2 因果链簇（事故→防御→backstop 因果链）
- 簇内 EK: EK-24（zed-industries/cloud 2026-05-18 事故）→ EK-28（initialHead 不变量）→ EK-23（mutates 派生门控）→ EK-26（native FS deny）
- 边类型: causal（事故→各层防御）
- 簇成立: 完整因果链（事件→第一层工具面门控→第二层 native FS deny→第三层 runtime backstop）；可命名；可回溯

**模式**：当一个"越权写"事故被确认后，防御必须同时落在**多个独立层**上，且每层用不同的失效模型：hook 门控（调用前拦截，靠 agent 钩子契约）、工具内校验（运行时状态不变量，靠 repo 事实）、FS 层 deny（进程内文件系统访问，靠路径规则）。任何单层的绕过不构成越权（纵深防御）。配套铁律：**每层防御必须能独立失效并独立验证**（对抗测试 fsExfil/gitNativeWrite 各自断言一层）。

**Epistemic**: Validated Pattern（PullFrog 内完整实现 + 测试断言）；跨项目验证 pending。

---

## KO-03 · 凭证生命周期即安全边界：从"假定可渗漏"推导授权，从授权推导刷新与吊销（L4 Cognitive Model）

**aggregation_rule**: R2 因果链簇（授权假设→scope→刷新→吊销链）
- 簇内 EK: EK-07（四类 token）→ EK-08（git 可渗漏⇒contents:read/write）→ EK-11（刷新单飞 + re-mint 解药）→ EK-12（run 结束吊销）→ EK-10（gh 可打印⇒角色镜像阈值）
- 边类型: causal + contrast（EK-08 可渗漏 ↔ EK-09 不可渗漏）
- 簇成立: 完整链（分类→授权→生命周期→边界）；解释范围扩大

**一句话**：凭证的**可触达性**决定授权粒度，授权粒度决定刷新与吊销策略——"agent 能打印它"与"agent 只能经工具面用它"是两种完全不同的信任假设，必须分化成不同 token、不同 scope、不同生命周期。

**L4 稳定关系**：`可触达性（exfiltratable?）→ 授权粒度（scope）→ 生命周期策略（刷新/吊销）` 三者必须同源推导；若 token 可被 agent 打印，其 scope 就是整个安全边界（EK-10 注释："the gh token scope IS the security boundary"）。

**Epistemic**: Validated Pattern（项目内强证据）；跨项目验证 pending。

---

## KO-04 · 可打印凭证的 scope 收敛规则（L5 Methodology 雏形）

**aggregation_rule**: R3 不变量簇（可打印 ⇒ scope=边界 不变量）
- 簇内 EK: EK-10（ghToken 单独签发 + 角色镜像 + 阈值下不注册）、EK-09（mcpToken 不可渗漏 ⇒ defense-in-depth scope）、EK-33（opencode MCP 权限 deny-all + allow /tmp）、EK-13（external GH_TOKEN 降级为用户信任假设）
- 边类型: constraint（EK-09/EK-33 约束 EK-10 的安全上限）+ mechanism（都是"可触达⇒收紧"）
- 簇成立: 内聚（同一不变量）；跨实例（gh CLI、MCP 工具、opencode 插件 3 个暴露面）；可命名；可回溯

**可操作准则**（L5）：
1. 任何会被 agent 进程直接打印/读取的凭证，按其**最小必要 scope** 单独签发，绝不复用高权限 token。
2. 凭证的 scope 若无法推导成最小集（如用户自选 scope 的外部 token），显式降级该路径的安全模型并在文档中声明。
3. 工具面的 token 权限按"即使工具上下文被攻破也够不到 secrets/admin"设计（defense-in-depth scope）。

**Epistemic**: Hypothesis（3 实例支持，尚未跨项目验证）→ 标 `Cross-project validation pending`。

---

## KO-05 · 双超时 + 噪声过滤 + safety-net：agent 失联治理的三段式（L4 Cognitive Model）

**aggregation_rule**: R4 主题簇（失联治理主题，互补维度）
- 簇内 EK: EK-37（outer 900s + first-event 120s）、EK-38（ACTIVITY_NOISE_PATTERNS 防 zombie）、EK-39（safety-net：先 dispose MCP 再 forceReject）、EK-35（SSE abort signal #876）
- 边类型: causal（事件活性→watchdog→safety-net）+ mechanism（超时族共享"agent 失联"问题）
- 簇成立: 主题内聚（失联治理）；互补维度（活性定义/噪声识别/终止顺序/event 语义）；可命名；可回溯

**一句话**："agent 失联"不是一个超时问题，而是**活性定义 + 噪声排除 + 终止顺序 + 失败再确认**四个问题：把代理自身的重连噪声从活性信号中排除（防僵尸）；把终止设计成"先断工具面、再终止 agent、再给回收机会"的顺序（防 re-prompt 落在死 MCP 上）；在判失败后仍给一个 safety-net 窗口让恢复中的 agent 自救。

**L4 稳定关系**：**失联判定 = 活性信号 − 噪声**；**终止语义 = 工具面失效先于 agent 失效**。

**Epistemic**: Validated Pattern（#12 zombie、#876 假 stalled、#1085 死 MCP re-prompt 三个事故闭环）；跨项目验证 pending。

---

## KO-06 · 可见交付门控：run 的完成 = 有可见产物的完成（L4 Cognitive Model）

**aggregation_rule**: R4 主题簇（交付门控主题）
- 簇内 EK: EK-40（四类 post-run gate）、EK-41（Review 只认 create_pull_request_review）、EK-29（diff-coverage pre-flight 一次 nudge）、EK-42（对抗测试把属性变断言）
- 边类型: mechanism（都是"交付前校验"）+ dependency（EK-40→EK-41）
- 簇成立: 主题内聚；互补维度（出口契约/覆盖校验/预算控制/可测试性）；可命名；可回溯

**一句话**：托管 agent 的 run 是否成功，不由"agent 正常退出"决定，而由"PR 上是否有可见产物"决定——Review 模式只认 review 提交（comment 不算）、软门一次性 nudge 不烧预算、硬门耗尽预算转 hard-fail（"run 交付了无可见产出的东西 = 失败"）。

**L5 行动准则**（雏形）：
1. 每个模式声明**唯一合法出口**（提交 review / 写 summary / 留 comment），出口校验失败即为 run 失败。
2. 软门（提示性）与硬门（失败性）分离；软门重试有预算且一次性，防 agent 靠 nudge 刷预算。
3. 对不可写面的 repo，宁可 burn retries 变红，也不绿-with-nothing。

**Epistemic**: Validated Pattern；跨项目验证 pending。

---

## KO-07 · 沙箱纵深而非单点：mount 命名空间内的秘密遮蔽与代码执行面封死（L3 Pattern）

**aggregation_rule**: R1 机制簇（沙箱机制跨多面）
- 簇内 EK: EK-15（三级探测）、EK-17（userns 不挂 --mount-proc + 双层锁）、EK-18（SOCKET_CLEANUP）、EK-19（FS_MOUNTS 三件套）、EK-22（ASKPASS git auth）
- 边类型: mechanism（都是"隔离/遮蔽"）+ causal（EK-18→EK-19）+ dependency（EK-19→EK-20）
- 簇成立: 内聚（同一隔离机制族）；跨实例（secrets 遮蔽/env 注入阻断/代码执行面只读/socket 封堵 4 面）；跨 ≥2 独立子系统（shell.ts 沙箱 + gitAuth.ts + nativeFsDenies.ts）；可命名；可回溯

**模式**：容器/mount 命名空间沙箱的正确用法不是"单点隔离"，而是**按逃逸面逐面封堵**：①on-disk secrets 用 tmpfs 遮蔽（codex auth.json）②env 注入用 tmpfs 覆盖 runner_file_commands（GITHUB_ENV 是后续 step 的代码执行通道）③git 代码执行面（filter/hook/config）整体 ro-bind ④容器 socket（docker.sock）bind /dev/null ⑤认证凭据不进子进程 env（ASKPASS 经唯一脚本文件）。并配套：**可探测降级链**（unshare→sudo-unshare→userns）而非单一实现，CI 下全失败即硬失败。

**Epistemic**: Validated Pattern（fsExfil 9 checks 断言 + tokenExfil passOnTimeout）；跨项目验证 pending。

---

## KO-08 · 执行面与决策面解耦：agent 系统安全的核心结构（L4 Cognitive Model，Core KO）

**aggregation_rule**: R3 不变量簇（解耦不变量汇聚）
- 簇内 EK: EK-07/08/09（token 分层：谁执行/谁决策分离到不同 token）、EK-23/26（subagent 与 orchestrator 工具面分离 + native FS deny）、EK-40/41（决策产物与执行产物分离校验）、EK-27（shell=disabled 下 git 工具面是唯一 exec 通道——执行面收敛）
- 边类型: constraint（各 EK 汇聚到"执行与决策必须解耦"不变量）+ mechanism
- 簇成立: 内聚；跨实例（凭证/工具面/交付面 3 维）；可命名；可回溯

**一句话**：**执行面与决策面解耦**——决策者（agent/模型）拥有判断力，执行面（工具/token/沙箱）拥有权威；判断力可以自由，权威必须最小化并按可逆性×影响授予。PullFrog 的三个实例：token 分层（决策用 mcpToken 不可渗漏，执行用 gitToken 可渗漏但最小 scope）、subagent 门控（子代理可读仓库但不能改工作树）、交付门控（agent 可以"决定"评论，但"提交 review"有唯一出口校验）。

**纵向链**：EK-08/09（L1）→ EK-23/26（L2）→ KO-02（L3 Pattern）→ KO-08（L4）——每一层均可回溯。

**Epistemic**: Core KO（跨 3 维强证据）。与用户此前验证过的 "Intelligence ≠ Authority"（Tafcm Untrusted Analyzer → P2）构成同构认知——见 05_candidates C-08。

---

## KO 配比检查

| 项 | 目标 | 实际 |
|----|------|------|
| KO 数 | 7~12 | 8 |
| 聚合规则覆盖率 | 100% | 8/8（R1×2, R2×2, R3×3, R4×2） |
| 簇平均规模 | 3~12 EK | 4.6（3~8） |
| 无悬空升维 | 强制 | 全部可回溯 EK 簇 |
| 无"同子系统=理由" | 强制 | 无 |
