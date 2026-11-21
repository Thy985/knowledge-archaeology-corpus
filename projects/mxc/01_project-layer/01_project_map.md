# 01 · Project Layer — MXC 项目地图

## 定位
- **产品**: Microsoft eXecution Container（MXC）——策略驱动的跨平台沙箱代码执行系统（GA 1.0.1，MIT）
- **核心命题**: "An agent cannot be its own security authority. It must run within a boundary defined by the developer or organization and enforced independently of the agent itself."
- **形态**: SDK 依赖（Rust/.NET/Node），进程内运行；或非 SDK 执行器 `wxc-exec.exe` 接受 JSON 容器创建请求

## 架构（两层：SDK facade → 引擎）
```
应用（Launch API）→ MXC SDK（Rust/.NET/Node，进程内）→ 后端（进程内）→ 隔离工作负载
                                     │
                        mxc_engine（后端选择/平台探测/执行，单一实现）
                                     │
        ┌───────────┬───────────┬────────────┬─────────────┬──────────────┐
        ▼           ▼           ▼            ▼             ▼              ▼
   Bubblewrap     LXC        Seatbelt   ProcessContainer  WSLC     IsolationSession
   (Linux)      (Linux)     (macOS)    (Win AppContainer/  (Win WSL)  (Win 独立会话)
                                        BaseContainer 3 层 fallback)
```
- **关键分层**: `mxc_common`（跨平台基础，不含后端依赖）→ `backends/*`（cfg 按目标平台拆分）→ `mxc_engine`（后端选择唯一所在地，依赖 backends）

## 核心模块（src/mxc-sdk/src/）
| 模块 | 职责 | 关键文件 |
|---|---|---|
| `core/mxc_engine` | 后端选择/平台探测/执行（唯一实现点） | dispatch.rs（streaming 分发）/ run.rs（run-to-completion）/ state_aware.rs（生命周期）/ mod.rs（7629 行总） |
| `core/mxc_common` | 跨平台基础：模型/沙箱进程/策略身份/审计/校验 | policy_identity.rs（PolicyHash 1205 行）/ audit.rs / sandbox_process.rs / validator.rs / filesystem_canonical.rs |
| `core/mxc_contract` | 版本化契约（published/development 分离） | registry.rs / version.rs（probe_version 精确匹配） |
| `core/learning_mode_core` | 学习模式抽象（DenialAnalyzer trait + DeniedResource） | analyze.rs / model.rs / capture_diagnostics.rs |
| `core/learning_mode_windows` | Windows 学习模式（sealed ETW .etl 解码） | etl_decode.rs / etl_filter.rs / tdh_decode.rs / process_lifetime.rs |
| `policy/` | 策略类型（authoring，非 JSON 契约） | network.rs / sdk_v1_conformance.rs / exact/ |
| `backends/` | 各平台后端 | bubblewrap/common/bwrap_command.rs、process_container/common/dispatcher.rs（3 层 fallback）、lxc/common/、seatbelt、isolation_session、hyperlight、nanvix |
| `state_aware_sdk.rs` | 有状态生命周期 API（provision/start/execute/stop/deprovision） | 生命周期五阶段 |
| `telemetry.rs` | 遥测同意管理 | ConsentState（Granted/Denied/Undetermined） |

## 核心数据结构
- **`ContainerRequest`**（v1 SDK authoring 类型）: command + filesystem/network/ui 策略 + timeout_ms + containment + container_name + working_directory + environment
- **`Containment`** 枚举: Process / Bubblewrap / Lxc / Seatbelt / ProcessContainer / Wslc / IsolationSession（Process 解析到宿主原生后端）
- **`ExecutionRequest`**（内部规范化模型）: 引擎消费，SDK 与 JSON parser 都归一到这里
- **`ContainerPolicy`**（内部策略类型）: filesystem/network/ui/timeout_ms，**省略字段=最严格（default-deny）**
- **`StateAwareOperation`**: Provision(StateAwareProvision) / Start{sandbox_id} / Stop{sandbox_id} / Deprovision{sandbox_id}
- **`DeniedResource`**: resource + resource_type（File/Network/UI…）+ access_type——学习模式统一模型
- **`BackendRegistration[10]`**: backend + experimental + probe_coverage（静态注册表）

## 状态与生命周期
- **一次性（one-shot）**: `v1::run`（run-to-completion）/ `v1::spawn`（streaming handle）
- **有状态（state-aware）**: Provision → Start → Execute → Stop → Deprovision（sandbox_id join key）
- **Phase 相关**: telemetry correlation_state 用 pre_dispatch_vector(active, phase, sandbox_id)

## 测试体系
- 352 个含 `#[test]` 的 Rust 文件（SDK api_tests / backends 各 runner 测试 / policy 测试 / state_aware_binding_tests）
- `tests/` 顶层: configs / examples / fixtures / playground / policy（sdk-v1 一致性 JSON）/ regression_tests（PowerShell 驱动）/ scripts
- `src/testing/`: wxc_test_driver / wxc_test_proxy / unix_test_proxy / wxc_ui_probe / fuzz

## 配置与治理机制
- **JSON 契约版本化**: `schemas/stable/mxc-config.schema.{0.4~0.9-alpha,1.0.0}.json` + `schemas/dev/1.1.0-alpha` + `schemas/schema-version.json`（min/maxSupported/stableLatest/stateAware/sdkMajorTargets）
- **三运行模式**: Enforcement（阻断）/ Learning（阻断+记录 JSON 活动报告）/ Permissive（允许+记录，用于起草最小权限策略）
- **双层策略**: workload 声明 + 组织 Intune 管理策略叠加（docs/containment-policy 语义）
- **PolicyHash 审计**: `mxc.PolicyHash` 记录每次运行的规范化策略身份
- **遥测同意**: 显式 consent（Windows 主机），非 Windows 不采集
- **SECURITY.md / SUPPORT.md / CONTRIBUTING.md** 治理文档

## 权限与治理机制（安全关键）
- **实验性 opt-in 是调用方参数**（experimental_enabled），不是 JSON 字段——配置不能自我授权（experimental.rs: require_experimental_optin）
- **Bubblewrap 网络拒绝**: ingress.default='allow' / hostLoopback='allow' 被显式拒绝（slirp 无名路由）；proxy 不能与直接 egress 规则混用（规则会被静默丢弃）
- **路径规范化**: canonicalize_path 用 GetFinalPathNameByHandleW 折叠 symlink/junction/8.3/\\?\ 别名
- **ProcessContainer 三层 fallback**: Tier1 BaseContainer → Tier2 AppContainer+BFS → Tier3 AppContainer+DACL；DACL tier 的 Drop 恢复宿主 ACE

## 外部依赖
- crates.io `mxc-sdk`（Rust）；NuGet `Microsoft.Mxc.Sdk`（.NET）；npm `@microsoft/mxc-sdk`（Node）
- 平台原生后端: Bubblewrap / LXC / AppContainer / Seatbelt / WSL / Hyperlight / Nanvix(MicroVM)
- Windows: ETW（学习模式 WPR 捕获）、AppContainer API、DACL

## 事实来源
以上全部来自 HEAD `fda29ac` 仓库实际内容（src/Cargo.toml workspace 成员、src/mxc-sdk/src/*、schemas/、docs/、tests/）。
