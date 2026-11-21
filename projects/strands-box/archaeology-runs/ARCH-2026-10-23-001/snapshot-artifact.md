# Repository Snapshot Artifact — ARCH-2026-10-23-001（strands-box）

## 仓库定位
- **repository**: https://github.com/strands-agents/box.git（AWS 官方开源线，Apache-2.0）
- **HEAD**: `8c46dacd5825ac1d11c6d1f4200f0daa657c5681`（main，2026-10-22 10:51:56 -0700，feat(policy): reload an authored policy over a live authority (#122)——考古前 1 天活跃）
- **version**: 0.1.0（crates/box/Cargo.toml；developer preview 2026-10-07，无 CHANGELOG）
- **规模**: 861 文件 / 544 Rust / 227,393 行
- **定位**: "An open source sandbox engine for AI agents"——OS 隔离（macOS Seatbelt/Linux namespace）+ Dogwood 语义策略（box.toml + policy.dw），harness agnostic，决策默认拒绝

## 基础地图（全部可回溯）

### 语言/技术栈
- Rust（Cargo workspace，resolver="2"）；rust-toolchain.toml 钉工具链
- 外部决策引擎精确 pin（安全 pin 契约）：
  - `cedar-policy = "=4.11.0"`（**安全 pin**：schema-directed context parsing 跨 4.x minors 不同，pin 是 fail-closed 契约一部分，非便利下限；Cargo.toml 注释原文）
  - `dogwood-language = "=1.0.0"`（"The policy engine. The ONLY decision engine."）
  - `dogwood-local-engine = "=1.0.0"`（单机 temporal 引擎：durable assembly 追加事件 + 从私有 history 文件恢复 monitor 状态）

### 入口
- `strands-box` CLI：crates/box/src/main.rs + command/（cli.rs / run.rs / policy.rs）；核心命令 `run`（394 行 run.rs：LockedRun/acquire_run_lock/policy_subject/启动 announce credential stores）与 `policy generate-schema`
- AGENTS.md 声明的**冻结面**：CLI verbs `run` + `policy generate-schema`；9 个 box.toml 顶层 key（name/box_dir/policy/agent/tool/egress/mcp/telemetry/experimental-features），name+box_dir 必填；[agent]/[tool.<name>] 5 字段（command/workspace/env/filesystem/native-filesystem-policy）

### 核心模块（workspace 7 crates）
| crate | 定位（Cargo.toml description 原文） |
|---|---|
| strands-box（crates/box） | "The standalone workload-only Strands zero-trust box CLI." |
| strands-box-policy（crates/policy，25,925 行） | "The local Policy Decision Point (PDP) — one in-process, fail-closed policy authority opened through a concrete facade." |
| strands-box-containment（crates/containment） | "Fail-closed process containment via private platform-native backends." |
| strands-box-credentials（crates/credentials） | "The credential vault: resolves each real secret and attaches it to an outbound request at the network boundary, so the workload never holds it." |
| strands-box-egress-gateway（crates/egress-gateway） | "The egress gateway — the outbound boundary of the zero-trust box. Provides mechanism controls, credential injection, and a domain-owned effect-interception seam." |
| strands-shell（crates/shell） | "A virtual shell sandbox for AI agents" |
| strands-box-telemetry（crates/telemetry） | "The box's own decision records, one lane per box." |

### 核心数据结构
- **Request enum**（crates/policy/src/request.rs:142）：PEP 递给 PDP 的唯一授权问题——Fs（ApprovedPath + FsOperation）/ Connect（host/ip/port，L4 持久 floor）/ Http（method/path/body_bytes/intercepted）/ McpCall（server/method/tool/prompt/uri/arguments）/ ShellExec（command/program/args/cwd——program 为**解析后**程序，alias 使二者不同且 alias 不单独决策）/ ShellSpawn（"Permitting this grants more than it appears to"——二进制直接对内核运行，无 fs:* 决策）
- **ApprovedPath**（type 携带 claim，无公开构造器——raw spelling 编译不过，而非"曾未运行的检查"）
- **Decision enum**（decision.rs:61）：Allow{rule, attribution, resource} / Deny{reason: DenyReason, rule, attribution, resource}；**DenyReason** = Forbidden（显式 forbid 命中）/ NoMatch（无 permit 匹配）/ PolicyPending（策略包未安装）/ InternalFault
- **PolicyAttribution**：token / rule / authored @id / @description（403 回执带 rule id+description 的机制基础）
- **PolicyEngine**（policy.rs:56）：open/open_staged/validate/validate_staged/reload（live authority reload——HEAD 提交点）/decide/record/effective/warnings/refine_tool_call；`SELF_DEFENDED_FILES = ["box.toml", "policy.dw"]`
- **GovernedBox / Operator / Principal / FsAccess / FsOperation**（schema.rs / request.rs）
- **containment model**（model.rs）：Operation / Scope / PreparedFilesystemPath / Network / SignalMode / ProcessInfoMode / IpcMode / BackendOverride（657 行枚举面）
- **egress capability**（capability/）：CapabilityOutcome → decide_request 折叠 → Verdict{Allow{mutations}/Deny{Credential|MutationCollision}}——机制失败不出网；跨能力 mutation 冲突 deny（finish_allow: "conflicting mutations to the same target — a config collision"）
- **credentials model**：Vault / Attach（Inbound/Outbound/Redactions）/ Destination/DestinationPattern/InjectMode/Locator/PhantomCheck/RouteSpec；sources::sign_request（AWS SigV4 在 source 层）；2026-08-07 公开面收窄重构（audit Emitter/Secret/PhantomToken 等转 pub(crate)）

### 状态与生命周期
- `box run`：prepare（LockedRun + acquire_run_lock——并发保护）→ hosted → 启动 announce credential stores/operation → agent 进程受 containment + policy 双重约束 → 决策记录进 telemetry
- PolicyEngine 状态：open（加载+校验）→ decide（每操作）→ reload（live authority 热重载，HEAD #122）；SchemaStage（policy 分段装载）
- containment backend：macOS（macho.rs + seatbelt.rs）/ Linux（namespace：authority/netns/probe/reaper/syscall/view）；contain_trampoline bin（args/main）

### 测试体系（多层级）
- 单元/集成：policy tests 29 个（fail_closed_guards / denial_messages / granular_fs_actions / load_time_rule_checks / cross_boundary_* / box_resource_scope / egress_denial_messages）+ containment tests 12 个（backend_seatbelt / child_ipc / contains_exec_target / gateway_fail_closed_linux / inherited_handles / lifecycle / preflight_new_cells / profile_conformance）+ egress/credentials 各自 tests
- **test-workload**：20 个 workload 套件（baseline/git/mcp-stdio/monty/node/python/rust/shell/native-filesystem/kill-claude/kill-codex/resume-claude/resume-codex/budget/hook/network-egress/manual…）——驱动 **built** strands-box 二进制（黑盒确定性套件）
- **test-integ**（workspace 显式 exclude）：black-box deterministic containment suite，驱动 built binary 子进程，自带 Cargo.lock（"excluded deliberately, not overlooked"）
- test-common / test-native-filesystem（读 built box 产物）
- CI：11 个 workflows（ci / full-tests / rust-test-lint / workload-suite / containment-deterministic / native-filesystem / verdict-hermetic / pr-title / strands-command / deploy-box-artifact / auto-strands-review）
- 性能约束（tenets）：policy <1ms（"asserted, not yet benchmarked"——诚实标注）/ startup <50ms macOS/Linux 进程沙箱 / MicroVM ~125ms / memory <20MB

### 配置
- `box.toml`（9 冻结顶层 key；agent/tool 进程规格；filesystem 列表；egress/mcp/telemetry 配置；experimental-features）
- `policy.dw`（Dogwood 规则：permit/forbid + when + temporal 算子）
- 凭证配置：VaultConfig + credsd_preflight + 5 种注入（Bearer/Custom header/Basic/Query param/AWS SigV4）
- 环境：~/.strands-box/b/ 命名空间（stop/ls/rm/reset 因 box_dir 由 caller 提供而删除）

### 权限与治理机制
- **Interface-frozen**（AGENTS.md STOP 段）：接口变更 + 基础前提变更需用户显式批准，agent 不得自批；feature-critic 视 BLOCKER 级
- commit/PR 流程：Problem/Solution/Tests 三节式 commit message；简化技术英语（ASD-STE100）；不 rewrite reviewer 已见的 history；PR 描述按 .github/PULL_REQUEST_TEMPLATE.md
- ADR 载体：docs/design/decisions.md（**145 个 anchor**；"The code is the only authority"——条目落后时以代码为准）
- Tenets 7 条（assume agent compromised / agent never holds the secret / secure by default, workable by default / log the decision not just the event / portable in definition, honest in guarantees / extensible at edges, fixed at core（policy engine 永不可换）/ performance is a constraint）
- SECURITY.md（漏洞披露）/ CODE_OF_CONDUCT / CONTRIBUTING / THIRD-PARTY-NOTICES

### 外部依赖
- 决策引擎（pin 见上）；monty/monty-types（Python 解释器适配，optional，不 fork——"the integration needs no interpreter change"）；strands-box-egress-gateway（adapter 目标）；rustls/rcgen（dev，TLS MITM 测试）
- .agents/pocs/（POC 代码，exclude 出 workspace——sandbox-ebpf 仅 bpfel-unknown-none target + nightly）

## 考古焦点（三主张）
1. **fail-closed 契约的工程化落实**：PolicyPending/InternalFault 拒绝语义 + cedar 安全 pin + SELF_DEFENDED_FILES + egress MutationCollision fail-closed + ApprovedPath 类型化路径——"fail-closed"不是口号而是每层机制
2. **凭证外挂（agent never holds the secret）**：credentials Vault + egress capability 注入 + 5 种注入模式 + sign_request——与引擎 eval 层"凭据注入白名单"同构（雷达卡已验证）
3. **时间性策略 + 统一事件语义**：dogwood-local-engine durable history + Request 跨工具归一（fs:read/http:request 统一上报）+ 403 回执带 rule id/description（agent 可自纠）
