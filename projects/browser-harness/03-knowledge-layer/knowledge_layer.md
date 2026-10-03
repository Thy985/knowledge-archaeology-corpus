# 03 Knowledge Layer — browser-harness（8 KO · 窄尖顶）

> Generalized Knowledge（L3/L4/L5）。每个 KO 声明 aggregation_rule（R1-R4）+ 簇内 EK + 边类型。簇成立五条件：内聚性/跨实例性/解释范围扩大/可命名/可回溯。升维候选全部经反例压力（见 06）。

## KO 聚合规则矩阵
| KO | 规则 | 簇内 EK | 边类型 | 解释范围扩大论证 |
|---|---|---|---|---|
| KO-01 | R1 机制簇 | EK-18/19/20/21/22/23 | mechanism（身份验证机制跨 _ipc/admin 两子系统） | "进程身份必须端到端验证"从 IPC 层扩散到 daemon 生命周期层 |
| KO-02 | R2 因果链簇 | EK-06→43→10→42 | causal（会话失效→恢复注册→重连→锁序） | 单个 CDP 错误展开为完整自愈故事 |
| KO-03 | R3 不变量簇 | EK-14/25/12/13/22 | constraint（permission 纪律约束所有连接路径） | "被拒不重连"是跨连接路径的安全不变量 |
| KO-04 | R4 主题簇 | EK-01/02/03/04/05/09 | subsystem（自愈架构） | heredoc+注入+中间人+追踪=一个完整架构模型 |
| KO-05 | R4 主题簇 | EK-24/26/27/32/33/38 | subsystem（cloud 授权与计费） | 云端浏览器的生命周期是一个完整治理主题 |
| KO-06 | R1 机制簇 | EK-28/29/30/31/36 | mechanism（脱敏机制跨日志/录制/遥测/视频四层） | "输出面统一脱敏"是同一机制的多实例 |
| KO-07 | R4 主题簇 | EK-05/09/11/15/16/17/40 | subsystem（后台操作原则） | 不打扰用户的浏览器控制是完整行为主题 |
| KO-08 | R2 因果链簇 | EK-02→31→34→35→36 | causal（观察→记录→编译→导出） | 观察钩子到视频产物的完整流水线 |

---
## L3 Patterns（可复用模板）

**P-01（KO-04 实例）Agent harness 的"受控中间层 + 运行时能力扩展"模式**
守护进程持有外部资源（浏览器会话）的权威；agent 进程经 IPC 请求动作；agent 的自定义能力以文件形式落在独立工作区，运行时注入。同类对照：lint 的受控接口、CI 的 checkout 沙箱、MCP server 模式。
证据：EK-01/03/04；helpers.py:655-668；AGENTS.md
Epistemic：Pattern（本项目强证据，跨项目验证 pending）

**P-02（KO-01 实例）进程身份验证模式（Reconciliation E-03 范围限定）**
对"可能被复用的 PID/端口/文件"做端到端身份确认：**ready daemon（有 IPC socket）路径** = 活体应答（ping 协议）→ 自报身份（PID）→ 起始时间指纹（start-time）三重验证后才允许发信号；**pending daemon（handshake-wait，无 IPC socket）路径**走 fingerprint generation 匹配（原子发布的进程起始指纹），owner 变化则不动手。两类路径共享原则：身份未端到端确认绝不发信号。同类：sshd 的 host key、consul 的 session 失效检测。
证据：EK-19/20/21/22 + EK-46 邻近的 pending 路径；_ipc.py:96-145；admin.py:760-880
Epistemic：Pattern（本项目强证据，跨项目验证 pending）

**P-03（KO-03 实例）"被拒不自动重试"的交互纪律模式**
对"需要人类批准才能继续"的交互（权限弹窗/登录墙）：拒绝或超时后绝不自动重试/创建替代连接——保持现场等人类决定；自动重试会把一次批准变成无尽弹窗。同类：sudo 超时后重新询问、OAuth 授权失败不静默重放。
证据：EK-14；admin.py:683-706；SKILL.md "Login walls: stop and ask"
Epistemic：Pattern（本项目强证据，跨项目验证 pending）

---
## L4 Cognitive Models（稳定关系，一句话）

**CM-01（由 P-01 升维）会话权威与任务能力分离**
Agent 对真实系统的控制 = 权限契约：会话权威（谁握着连接/状态）与任务能力（agent 会写什么代码）是两个独立维度，前者由守护进程持有、后者可在独立工作区自由增长；能力增长不自动扩展权威。
证据：EK-03/04/05 + AGENTS.md（核心 src 受保护）
升维论证：解释范围从"浏览器 harness"扩到"任何 agent-外部系统边界"（权限边界 = EP-002 关注点）。
Epistemic：Cognitive Model（cross-project validation pending——browser-harness 单项目证据）

**CM-02（由 P-02 升维）信任 = 端到端可验证的身份，而非文件/端口的存在**
PID 文件、端口、socket 文件的存在都不构成信任；对恶意/陈旧/复用的中间态，信任必须由"端到端应答 + 身份 + 起始指纹"重建。声明 ≠ 权威。
证据：EK-18~22；_ipc.py ping/identify 全部防御分支
Epistemic：Cognitive Model（cross-project validation pending）

**CM-03（由 P-03 升维）交互性拒绝是状态，不是错误**
人类批准类的拒绝/超时是待处理状态（等人类），不是可自动重试的瞬态错误（可退避）。把两者混淆会把系统推进"自动制造更多弹窗"的灾难循环。
证据：EK-14 的修复史注释（"how a single approval turned into an endless prompt"）
Epistemic：Cognitive Model（cross-project validation pending）

---
## L5 Methodology（可操作准则）

**M-01 Agent 影响真实外部状态的动作必须经守护中间层**
动作经守护进程（持有会话/计费/权限权威）转发；agent 直接触碰真实状态的路径越少越好。可执行：让 harness 类工具持 CDP/会话，agent 只持任务与工作区文件。
证据：EK-04/05/18 + SKILL.md 后台操作原则
Epistemic：Methodology（browser-harness 验证；跨项目 pending）

**M-02 Agent 自扩展能力须限定在独立工作区且经显式加载点接入**
新增能力以文件落地（agent_helpers.py/domain-skills），经唯一加载点（_load_agent_helpers）注入受控命名空间；核心库不改、接口不扩。可执行：为 agent 定义"扩展目录 + 加载点"而非开放 import。
证据：EK-03/41；helpers.py:655-668
Epistemic：Methodology（browser-harness 验证；跨项目 pending）

**M-03 对不可信 agent 的输出面统一脱敏，且脱敏必须发生在持久化/外发之前**
日志/录制/遥测/视频四层各持独立脱敏正则（拓扑/URL 参数/键名/敏感实体），且位于写盘/上报之前。可执行：外发前检查链上每一层是否"输出面清洗"。
证据：EK-28/29/30/36；recorder.py:52-67
Epistemic：Methodology（browser-harness 验证；跨项目 pending）

---
## 与 02 的反向追溯
- 每个 KO 的每条 EK 均可回溯：EK-01~45（02 文件）均有 Evidence 行号。
- 无悬空升维：P-01~03 由 EK 簇直接导出；CM/M 均锚定具体 EK 与证据。
- 反例预算：每个 L3+ 候选 ≥3 定向反例攻击，见 06_validation.md §反例。
