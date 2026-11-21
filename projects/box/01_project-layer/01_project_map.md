# Project Layer — Strands Box 项目地图

## 是什么（README 原文）
"Strands Box is an open source sandbox for AI agents. It combines operating-system isolation with semantic policies written in Dogwood, so you can control what agents can access and the conditions under which they can act."

## 架构（architecture.md 权威）
**三部分**：agent 沙箱 + 每工具/每本地 MCP server 独立叶沙箱 + box 可信进程（`strands-box run`，沙箱外）。
**双层强制**（containment.md:20-40）：
| | OS enforcement | Policy |
|---|---|---|
| 覆盖 | 进程自身 syscall（文件打开等） | agent 经可信进程发出的请求（Shell 命令/Monty 脚本/MCP 调用/出站网络） |
| 谁执行 | 内核（Seatbelt / Linux namespace） | 可信进程内四 enforcement point 问 Dogwood 引擎 |
| 来源 | box.toml（+系统路径） | policy.dw |
| 时机 | 进程启动前一次性 | 每个请求 |

**可信进程内组件**（architecture.md "The parts"）：
- Policy engine：唯一授权者，每 box 一个；持有跨 enforcement point 的共享历史
- Broker：run/box.sock 上接 alias 连接，三腿（Shell/Monty/MCP），自己零授权
- Strands Shell / Monty：可信进程内解释器（无状态跨调用），向 policy 问 `shell:exec`/`shell:spawn`/`fs:*`
- Egress gateway：唯一出站路线；`net:connect`（DNS 前+每地址再判）+`http:request`+`mcp:call`（远程 MCP）；凭据注入点；终止所有连接（无隧道透明）
- MCP broker：stdio server 启动要 `shell:spawn` permit，之后每请求 `mcp:call`
- Collector：决策记录（OTLP JSON），enforcement point 提交（effective decision 为 canonical）
- Trampoline：自遏制后 exec 目标程序（macOS 沙箱单向，run 从不自调用 sandbox）
- Alias：沙箱内唯一 Box 程序（zsh/bash/sh/python3/python/MCP server 名），无权力

## 进程拓扑图（mermaid 摘要见 architecture.md:150-205）
agent(沙箱) → alias → run/box.sock → broker → {Shell,Monty,MCP broker} → policy
agent(沙箱) → 所有出站 → egress gateway → policy；gateway 转发 + 凭据注入

## 生命周期（architecture.md "How a run comes up"）
run 定序 7 步（自硬化→读配置+run 锁+记录身份→开 collector→装 policy（拒解析失败/不可触发规则）→解析凭据+起 gateway（自签 CA）+判 fs reach+bind socket→构建沙箱 grants 并披露→起 trampoline→agent→等待）。
拆除反序：reap workload → broker unbind → gateway 收尾 → 撤运行记录 → 锁最后释放 → collector drain（export 存活于锁后）。

## 核心数据结构
- `CapabilityOutcome{Applied(Vec<Mutation>), Unavailable(CapabilityFault)}`（egress-gateway/capability/traits.rs）——无 Allow/Deny
- `Verdict{Allow{mutations}, Deny{reason}}` + `Mutation`（SetHeader/StripHeader/RewritePath/AddQueryParam/StripQueryParam，Zeroizing+Debug redacted）（boundary/verdict.rs）
- `ContractVersion`（dogwood 精确 pin）；box.toml 九顶层键（name/box_dir/policy/agent/tool/egress/mcp/telemetry/experimental-features）
- 文件系统 grants：agent 八列表 read/write/read_file/write_file/list/metadata/exec/deny；tool 六列表（无 metadata/exec）

## 测试体系
- test-integ/：黑盒确定性遏制套件（驱动 BUILT strands-box 子进程；独立 Cargo.lock；从 workspace exclude）
- test-workload/：workload-{baseline,budget,agent-hook}、network-egress、verdict
- test-native-filesystem/：原生文件系统策略（native file checks 逐 workload 选择——HEAD commit #113）
- test-common/：共享 reader；各 crate 内 #[test]（policy/egress/containment/telemetry 为主）
- 173 个测试 .rs 文件（test-* 目录合计）

## 配置
- box.toml：直接访问 grants + env（禁代理变量名）+ egress routes（env:// 或 aws://）+ mcp + telemetry + experimental-features
- policy.dw：Dogwood permit/forbid；缺省=全拒；deny-overrides

## 权限与治理机制
- **interface-frozen**（AGENTS.md）：CLI verbs/box.toml 九键/五字段/四输入/Record 版本/协议版本/动作词汇全冻结，改动需用户显式批准（agent 不可自我授予）
- **9 premises 各带测试钉住**：authored policy is the only authorizer；workload untrusted；validate once act on identity；credential binding is not authorization；HOME 是 operator 的（home grants no reach）等
- **7 tenets**（tenets.md）：Assume the agent is compromised / The agent never holds the secret / Secure by default, workable by default / Log the decision, not just the event / Portable in definition, honest in guarantees / Extensible at the edges, fixed at the core / Performance is a constraint

## 外部依赖
- dogwood_language + cedar_policy（policy crate pin 精确版本）
- strands-agents/shell（上游 shell crate）
- opentelemetry（collector/export/receive）
- macOS Seatbelt（当前平台）；Linux namespace（代码在，preview 声明 macOS only）
- zeroize（Mutation 凭据擦除）
