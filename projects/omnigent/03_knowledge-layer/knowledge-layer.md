# 03 · Knowledge Layer — Omnigent（Generalized Knowledge，窄尖顶）

> v3.1 规范：每个 KO 必须声明 `aggregation_rule`（R1 机制簇 / R2 因果链簇 / R3 不变量簇 / R4 主题簇）+ 簇内 EK + 解释范围扩大论证。禁止"同子系统=聚合理由"。所有升维标注认知状态：Pattern（L3）/ Cognitive Model（L4）/ Methodology（L5）。单项目证据 → 标注 `Omnigent-originated · Cross-project validation pending`。

## 03.0 KO 聚合规则矩阵

| KO | aggregation_rule | 簇内 EK（边类型） | 解释范围扩大 |
|----|-----------------|------------------|--------------|
| KO-01 | R4 主题簇 | EK-01/02/04/14（subsystem+causal） | 从"runner/server 分离"扩到"meta-harness 分层如何同时获得执行与协作" |
| KO-02 | R2 因果链簇 | EK-03→04→05→06→10（causal 链） | 从"策略在 runner"扩到"策略执行点跟随控制权迁移的完整机制故事" |
| KO-03 | R1 机制簇 | EK-04/08/09/10（mechanism：审批/双评估，跨 runner/server/pending_approvals） | 从单一 ASK 流程扩到"人机闸门的设计空间（等待/拒绝/幂等/超时）" |
| KO-04 | R4 主题簇 | EK-11/12/13/16（subsystem+mechanism） | 从 Claude bridge 扩到"异构 harness 适配如何不腐烂" |
| KO-05 | R1 机制簇 | EK-17/23/24（mechanism：会话/资源自愈，跨 bridge 目录/bundle/runner） | 从单点自愈扩到"大型 agent 系统的自愈模式族" |
| KO-06 | R3 不变量簇 | EK-18/19/21/22（constraint 汇聚到两个不变量：身份持久、凭据隔离） | 从沙箱管理扩到"代执行环境的安全不变量设计" |
| KO-07 | R2 因果链簇 | EK-25→26→29（causal：诊断→崩溃报告→观测审计） | 从单 bug 扩到"失败可解释性作为系统级投资" |

## 03.1 KO（L3 Pattern）

### KO-01 meta-harness 分层：执行面 / 控制面 / 适配面三分离
- **内容**：meta-harness 把"agent 会话执行"（runner）、"多租户控制与协作"（server）、"vendor 适配"（native bridge）分为三个正交面；组合、控制、协作由此都能独立演化。
- **回溯**：EK-01（runner/server/bridge 三件）、EK-02（分发归 runner）、EK-04（elicitation 归 server）、EK-14（Claude 桥实例）。
- **认知状态**：Pattern（L3）· Omnigent-originated · Cross-project validation pending。

### KO-02 策略执行点跟随控制权
- **内容**：当某项控制权（如 MCP dispatch）从 server 迁移到 runner，相应的策略执行点必须随之迁移，否则 parity 断裂；迁移后按能力边界（谁有 ConversationStore/LLM）决定剩余执行点归属。
- **回溯**：EK-03（RUNNER_MCP 决策）、EK-04（双评估）、EK-05（能力边界）、EK-06（组合语义）、EK-10（loud-failure）。
- **认知状态**：Pattern（L3）· 与候选卡 [cand]agent-harness-control-plane（"策略在 harness 层而非 prompt 层"）独立印证 · Cross-project validation pending。

### KO-03 人机闸门设计空间：等待 / 拒绝 / 幂等 / 超时
- **内容**：policy ASK 是 human-in-the-loop 闸门，设计空间含四个旋钮：默认等待时长（86400s vs 旧 120s）、fail-closed vs fail-open（headless 显式传有限 timeout）、等待注册表生命周期（register/cleanup(finally)/resolve 幂等）、拒绝的可见性（DENY 文本作为 tool output 返回）。
- **回溯**：EK-04/08/09/10。
- **认知状态**：Pattern（L3）· 与 deepseek-harness/rampart 的审批-权限主题同域（corpus 连接）· Cross-project validation pending。

### KO-04 harness 适配能力声明化
- **内容**：把 harness 的隐式能力（散落 if 分支 + 伴随模块存在性）提升为显式声明模型（IntegrationMode 5 种 + Elicitation 4 种 + capabilities 表），注册表即可直接回答"能做什么"，社区扩展走 entry point 无 import cycle。
- **回溯**：EK-11/12/13/16。
- **认知状态**：Pattern（L3）· Omnigent-originated · Cross-project validation pending。

### KO-05 会话/资源自愈模式族
- **内容**：alpha 大型 agent 系统对"会话与资源死亡"的三类自愈：bridge 孤儿收割（owner.pid + 保守 liveness）、bundle 缺失重传（启动自愈）、runner 重连（server 侧 relaunch 后自动恢复）。共同点：**把"可能死亡的东西"变成"可恢复的东西"**，且收割永远保守（拿不准就不删）。
- **回溯**：EK-17/23/24。
- **认知状态**：Pattern（L3）· 与 evolver 自我更新可恢复（corpus）同域 · Cross-project validation pending。

### KO-06 代执行环境安全不变量：身份持久 + 凭据隔离
- **内容**：云沙箱（代执行环境）的两个安全不变量：**身份绑定锚定身份而非资源实例**（host_id 持久、sandbox generation 可换、会话绑定存活）；**用户凭据永不进入代执行环境**（专用 launch token + secrets 走 sandbox env 引用）。外加回收治理（离线 reaper）与配置注入语义（server-managed 覆盖、用户配置存活）。
- **回溯**：EK-18/19/21/22。
- **认知状态**：Pattern（L3）· 与 dsh-memory-evolve 凭据/锁设计、rampart 沙箱主题同域 · Cross-project validation pending。

### KO-07 失败可解释性系统化
- **内容**：系统把"模糊失败"持续改造为"可归因失败"：wedged LLM → 单槽 connectivity 健康记录（#1119）；崩溃 → 三 chokepoint 崩溃报告 + 预填 issue；观测缺口 → OBSERVABILITY 诚实审计与 tracing 计划。共同特征：**诊断信号要防止误归属（时间戳 + 成功清槽），崩溃路径要全捕获（主/从线程 + segfault），观测现状要被书面承认**。
- **回溯**：EK-25/26/29。
- **认知状态**：Pattern（L3）· Omnigent-originated · Cross-project validation pending。

## 03.2 Cognitive Models（L4）

### CM-01 执行点跟随控制权
- **内容**：**判断与执行的耦合位置由控制权归属决定——控制权迁移时，门控/策略若不随之迁移就会失效。**
- **回溯**：KO-02（EK-03/04/05）。
- **认知状态**：Cognitive Model（L4）· 单项目强证据，跨项目验证中。

### CM-02 审批超时是产品决策，不是技术默认
- **内容**：**人机闸门的默认超时语义决定系统的"默认信任方向"——默认等待 = 默认信任人类会回来（fail-open），默认拒绝 = 默认不信任（fail-closed）；两者都是产品选择。**
- **回溯**：KO-03（EK-08 的 120s→86400s 修复史）。
- **认知状态**：Cognitive Model（L4）· 单项目强证据。

### CM-03 绑定锚定身份而非资源实例
- **内容**：**在资源易逝的环境中，持久绑定必须锚定身份（host_id），资源实例（sandbox generation）可替换而不破坏绑定。**
- **回溯**：KO-06（EK-18）。
- **认知状态**：Cognitive Model（L4）· 单项目强证据。

### CM-04 失败可解释性是可累积的工程投资
- **内容**：**系统的可靠性能力 ≈ 把模糊失败转化为可归因失败的能力；每次"为什么失败说不清"都是一次未完成投资。**
- **回溯**：KO-07（EK-25/26/29）。
- **认知状态**：Cognitive Model（L4）· 单项目强证据。

## 03.3 Methodologies（L5）

### M-01 适配层用显式能力声明替代隐式分支
- **内容**：**接入 N 个外部系统时，为每个系统的能力建立显式声明模型（模式枚举 + 能力表），拒绝"if name == x"分支累积。**
- **回溯**：KO-04（EK-11/12/16）。
- **认知状态**：Methodology（L5）· Omnigent 验证。

### M-02 凭据永不进入代执行环境
- **内容**：**任何委托执行环境（沙箱/远端 runner）必须用每任务 mint 的专用令牌认证，用户主凭据只存在于控制面。**
- **回溯**：KO-06（EK-19）。
- **认知状态**：Methodology（L5）· 与 rampart/dsh 锁设计同域印证。

### M-03 诊断信号防误归属：单槽 + 时间戳 + 成功清槽
- **内容**：**进程级诊断记录要满足：单槽假设（一个工作单元一个槽）、时间窗（只归因足够新的失败）、复位（成功清除旧失败），否则旧失败会污染后续诊断。**
- **回溯**：KO-07（EK-25）。
- **认知状态**：Methodology（L5）· Omnigent 验证。

### M-04 CLI 输出通道分离
- **内容**：**CLI 的 stdout 只承载可解析数据、stderr 承载装饰，并禁止手写 ANSI——保证管道场景 byte-clean。**
- **回溯**：KO-07（EK-30）。
- **认知状态**：Methodology（L5）· 行业惯例 + Omnigent 契约化。

## 03.4 三层配比

| 层 | 目标 | 实际 |
|----|------|------|
| Facts/Evidence | 100+ | ~40 条（含 01 层 + EK 证据） |
| Engineering Knowledge | 40~60 | 30（本规模按比例） |
| Patterns（KO） | 7~12 | 7 |
| Cognitive Models | 少量 | 4 |
| Methodologies | 少量 | 4 |

- **Reconciliation 补强（Auditor）**：EK-31 CredentialProxy（swap-on-access + `oa_cred_*` 占位符 + 跨 host 403 泄漏守卫）为"凭据隔离"提供实现级证据——沙箱内无凭据形状，真 secret 只存在于控制面，出口代理按 host 绑定注入；kubernetes.py:492 的 `configure_clone_credentials` 为代理绑定配置。原 NEEDS_HUMAN_REVIEW #1 关闭。

- **Reconciliation 补强（Auditor）**：EK-31 CredentialProxy（swap-on-access + `oa_cred_*` 占位符 + 跨 host 403 泄漏守卫）为"凭据隔离"提供实现级证据——沙箱内无凭据形状，真 secret 只存在于控制面，出口代理按 host 绑定注入；kubernetes.py:492 的 `configure_clone_credentials` 为代理绑定配置。原 NEEDS_HUMAN_REVIEW #1 关闭。
