# Snapshot Artifact — ARCH-2026-10-20-001（Strands Box）

## 仓库快照
| 项 | 值 |
|---|---|
| repository | https://github.com/strands-agents/box.git |
| branch | main |
| HEAD commit | `069c39cfd29f830aaadd2a1af362753a57c7b202` |
| HEAD message | "feat(box): choose native file checks for each workload (#113)" |
| 克隆方式 | shallow（--depth 1），2026-10-20 |
| 文件数 | 856 |
| 语言 | Rust（Cargo workspace，resolver=2） |
| License | Apache-2.0 |
| GitHub 元数据 | ★443；created 2026-10-02；pushed 2026-10-19；open_issues 50 |

## 项目定位（README 原文锚）
"Strands Box is an open source sandbox for AI agents. It combines operating-system isolation with semantic policies written in Dogwood." —— 策略驱动的 OS 级 agent 行为遏制沙箱（当前 macOS Apple silicon preview；Linux containment 后端代码已存在）。

## Cargo workspace（8 成员）
| crate | 职责（Cargo.toml description 原文） |
|---|---|
| box | "The standalone workload-only Strands zero-trust box CLI."（二进制无库目标） |
| credentials | "The credential vault: resolves each real secret and attaches it to an outbound request at the network boundary, so the workload never holds it." |
| policy | "The local Policy Decision Point (PDP) — one in-process, fail-closed policy authority opened through a concrete facade." |
| containment | "Fail-closed process containment via private platform-native backends." |
| egress-gateway | "The egress gateway — the outbound boundary of the zero-trust box. Provides mechanism controls, credential injection, and a domain-owned effect-interception seam." |
| shell (+strands-shell-macros) | "A virtual shell sandbox for AI agents"（独立上游 strands-agents/shell） |
| telemetry | "The box's own decision records, one lane per box." |
| exclude | .agents/pocs（POC 不入常规 build）；test-integ / test-common / test-native-filesystem / test-workload（黑盒套件，驱动 BUILT 二进制非链接 crate） |

## 入口与进程
- **入口**: `strands-box run <box_dir>`（crates/box/src/main.rs → command/cli.rs → command/run.rs）；`policy generate-schema`；`strands-box-contain-trampoline`（遏制 trampoline）；`strands-box-sock-alias`（沙箱内 alias）
- **进程拓扑**（architecture.md）: 一个 box = 一个可信进程 `run`（policy 引擎+四 enforcement point+broker+collector+CA+全部明文密钥）+ agent 沙箱 + 每工具/每本地 MCP server 独立叶沙箱 + trampoline（自遏制后 exec 目标程序）+ alias（沙箱内唯一 Box 程序）

## 核心模块与数据结构
- **policy crate**: `EffectInterceptor` seam；`CapabilityOutcome{Applied,Unavailable}`（无 Allow/Deny——能力不能授权）；cap.rs `judge()` 加载期拒绝惰性 temporal permit
- **egress-gateway**: `Verdict/Mutation/DenyReason`；Mutation 枚举 SetHeader/StripHeader/RewritePath/AddQueryParam/StripQueryParam（Zeroizing 擦除、Debug redacted）；capability/{credential,decider,host}
- **credentials**: attach/audit/backend/legs/model/{destination,inject,material}
- **containment**: backend/macos/seatbelt.rs（Seatbelt profile）；backend/linux/namespace/{authority,netns,probe,reaper,syscall,view}（Linux namespaces + syscall filter，非 Landlock——decisions.md:974）
- **box/record/config/**: box.toml 九顶层键（name/box_dir/policy/agent/tool/egress/mcp/telemetry/experimental-features）
- **telemetry**: collector.rs/record.rs/export.rs（OTLP JSON 决策记录，默认 `<box_dir>/private/telemetry/records.jsonl`）

## 状态与生命周期（run.rs 定序）
1. 自硬化（拒同用户进程读其地址空间）→ 2. 读 box.toml+取 run 锁（记录 canonical 路径+文件系统身份）→ 3. 开 collector → 4. 装 policy（解析/验证失败拒 box；**不能触发的规则拒绝加载**）→ 5. 解析凭据+起 gateway（自签 CA）+判定每个进程规格的 fs reach+bind broker socket → 6. 构建各沙箱 grants 并披露（stderr）→ 7. 起 trampoline→agent→等待。拆除反向：先 reap workload → broker unbind → gateway 收尾 → 撤运行记录 → 锁最后释放 → collector drain 导出（export 存活于锁后）。

## 测试体系
- **test-integ/**（黑盒确定性遏制套件）：驱动 BUILT strands-box 二进制子进程，独立 Cargo.lock，cargo build --workspace 排除（README 明示原因）
- **test-workload/**：workload-{baseline,budget,agent-hook} + network-egress + verdict + manual + common
- **test-native-filesystem/**：原生文件系统策略
- **test-common/**：共享 reader
- 单元测试：各 crate #[test]（决策测试集中于 policy/egress/containment/telemetry）

## 配置与权限治理机制
- **box.toml**: 直接访问 grants（[agent.filesystem] 八列表 read/write/read_file/write_file/list/metadata/exec/deny；[tool.<name>.filesystem] 六列表——无 metadata/exec）；环境表禁设代理变量名
- **policy.dw**: Dogwood（Cedar 超集+时间规则）permit/forbid；空或缺省策略=全拒；deny-overrides 组合；**无编译 tier**（decisions.md:595）
- **治理自约束**: AGENTS.md interface-frozen（外部可观察面冻结+前提变更需用户批准，agent 不能自我授予）+ 9 premises 各带测试钉住（"authored policy is the only authorizer / workload untrusted / validate once act on identity / credential binding is not authorization / HOME 是 operator 的"等）

## 外部依赖
- **dogwood_language** / **cedar_policy**（策略引擎，policy crate 内部 pin 精确版本）
- **strands-agents/shell**（上游 shell crate 仓库）
- 运行时：macOS Seatbelt（当前平台）；Linux namespace 后端（代码在、preview 声明 macOS only）；Node.js 22.21+（Strands CLI 侧）
- 凭据方案：env:// + aws://（SigV4）两种 route scheme（拒绝其他 scheme at load）

## 事实基线
- 全部事实来自上述 repo 实盘文件（README.md / Cargo.toml / AGENTS.md / docs/design/{architecture,decisions,containment,credentials,egress,policy,tenets}.md / crates/*/src / test-* 目录），HEAD `069c39c`
- 与已考古 corpus 连接：dogwood（策略语言本体 09-08）；MXC（同域对照 10-19）；deepseek-harness（sandbox/approval seam）；agent-governance-toolkit（无状态策略引擎）
