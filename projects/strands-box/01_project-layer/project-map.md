# 01 Project Layer — strands-box（项目地图）

## 一、是什么 / 怎么运行
- **仓库**: github.com/strands-agents/box（Apache-2.0）；version 0.1.0（preview）；HEAD 8c46dac（2026-10-22）
- **运行模型**: `box run` → 解析 box.toml（ConfigureRequest 验证一次）→ containment 遏制 agent 进程（Seatbelt/namespace）→ Shell/Monty/egress gateway/MCP broker 把各自处理的操作送内嵌 Dogwood 引擎 → 决策默认拒绝，permit 显式放行，forbid 覆盖 → 决策记录进 telemetry（private/telemetry/records.jsonl）

## 二、架构（双层 + 三拦截点）
```
box.toml + policy.dw（operator 声明）
        │
        ▼
┌─────── box run（strands-box CLI）───────┐
│  containment（OS 遏制：macOS Seatbelt / Linux namespace）│
│  ├─ Strands Shell（shell:exec/spawn + fs:*）│
│  ├─ Monty for Python（fs:*）│
│  ├─ egress gateway（net:connect/http:request + 凭证注入）│
│  └─ MCP broker（mcp:call，每工具 refine_tool_call）│
└──────────────┬────────────────────────┘
               ▼
      PolicyEngine（PDP，唯一决策引擎 Dogwood）
      决策 → DecisionObserver → telemetry records.jsonl
```

## 三、核心模块（workspace 7 crates，见 snapshot）
| crate | 职责 | 关键文件 |
|---|---|---|
| box | 零信任 CLI（run/policy generate-schema） | src/command/run.rs（394 行） |
| policy | 本地 PDP（fail-closed authority facade） | src/policy.rs, request.rs, decision.rs, authority.rs(1581 行), adapters/ |
| containment | fail-closed 进程遏制（平台原生后端） | src/backend/{macos/seatbelt.rs(2750),linux/namespace/*} |
| credentials | 凭证 Vault（边界解析+注入） | src/vault.rs, attach.rs, sources.rs |
| egress-gateway | 出站边界（capability fold + 凭证注入） | src/capability/{decider.rs,credential/}, boundary/ |
| shell | Strands Shell（虚拟 shell 沙箱） | 内嵌 Lua 5.4（mlua vendored） |
| telemetry | 决策记录（一 lane/box） | src/lib.rs（Collector/DecisionRecord） |

## 四、核心数据结构
- Request enum（Fs/Connect/Http/McpCall/ShellExec/ShellSpawn）——PEP→PDP 唯一授权问题
- Decision（Allow/Deny{reason}）；DenyReason = Forbidden/NoMatch/PolicyPending/InternalFault
- PolicyAttribution（token/rule/@id/@description）
- ApprovedPath（类型化路径，无公开构造器）
- PolicyEngine（open/validate/reload/decide/record/effective/warnings/refine_tool_call）；SELF_DEFENDED_FILES=["box.toml","policy.dw"]
- Vault/Attachment（Inbound/Outbound/Redactions）；CapabilityOutcome→Verdict{Allow{mutations}/Deny{Credential|MutationCollision}}
- Containment model（Operation/Scope/Network/SignalMode/IpcMode/BackendOverride）

## 五、生命周期
- box run：prepare（LockedRun+acquire_run_lock）→ hosted → announce credential stores → agent 运行（三拦截点受 PolicyEngine 约束）→ 退出
- PolicyEngine：open（加载+校验）→ decide（每操作，决策记录）→ reload（live authority 热重载 #122——调用方负责谁可触发，"nothing here authenticates a request"）
- 组件失败：机制失败 → Deny（Credential），无降级放行

## 六、测试体系
- 单元/集成：policy 29 + containment 12 + egress/credentials/shell 各自 tests
- **test-workload 20 套件**（黑盒驱动 built binary：baseline/git/mcp-stdio/monty/node/python/rust/shell/native-filesystem/kill-claude/kill-codex/resume-*/budget/hook/network-egress）
- test-integ（显式 exclude 出 workspace，自带 Cargo.lock）；test-native-filesystem
- CI 11 workflows（ci/full-tests/rust-test-lint/workload-suite/containment-deterministic/native-filesystem/verdict-hermetic/pr-title/strands-command/deploy-box-artifact/auto-strands-review）

## 七、配置
- box.toml（9 冻结顶层 key：name/box_dir/policy/agent/tool/egress/mcp/telemetry/experimental-features；name+box_dir 必填）
- policy.dw（Dogwood：permit/forbid + when + temporal）
- VaultConfig（5 注入：Bearer/Custom header/Basic/Query param/AWS SigV4）；~/.strands-box/b/ 命名空间

## 八、权限与治理机制
- interface-frozen（两类变更需用户批准，agent 不得自批；feature-critic BLOCKER 级）
- decisions.md 145 ADR anchor（"The code is the only authority"）
- SECURITY.md / CODE_OF_CONDUCT / CONTRIBUTING；.agents/skills（11 个 agent 技能：authoring-box-policy/setting-up-a-box/debug-os-failure 等）

## 九、外部依赖
- 决策引擎精确 pin（cedar =4.11.0 安全 pin / dogwood-language =1.0.0 唯一引擎 / dogwood-local-engine =1.0.0 temporal）
- monty/monty-types（Python 适配，optional 不 fork）；rustls/rcgen（dev，TLS MITM 测试）；Lua 5.4（mlua vendored 非可选）
