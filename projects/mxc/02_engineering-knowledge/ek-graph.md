# 02 · Engineering Knowledge — MXC EK Graph（EK-M01~M20）

> 每条 EK 声明 links（6 类边）。证据格式：`文件:行`，全部指向 HEAD `fda29ac` 仓库实际内容。

## EK 清单

### EK-M01 · 策略身份 = 规范化哈希（PolicyHash）
- **层**: L1/L2
- **内容**: `policy_hash` 对 `ExecutionRequest` 的**显式 allow-list 投影**计算 SHA-256，格式 `sha256:<hex>`；稳定于格式/键序/base64-vs-文件输入，敏感于每个 enforcement 相关字段。被排除字段有明确理由：`script_code`（命令行=运行内容非策略+常嵌凭据）、`env`（经典秘密载体）、`network_proxy.original_url`（可嵌 user:password@）、`capture_denials.output_path`、`dry_run`、`experimental_enabled`、`source_contract`、telemetry 等。**`policy_projection` 穷举解构 ExecutionRequest**——新增字段未分类为 hashed 或显式 excluded 则编译失败（防"漏哈希秘密"安全属性静默腐烂）。哈希确定性且无密钥 → 明确接受"残余确认预言机"性质（对覆盖字段暴力猜测），但排除了真正敏感输入。
- **证据**: src/mxc-sdk/src/core/mxc_common/policy_identity.rs:1-90（文档+算法标签）、:1205（穷举投影）
- **links**: causal→EK-M02（PolicyHash 发到 mxc.PolicyHash 审计记录）；constraint→EK-M07（审计记录禁配置值）；mechanism↔EK-M20（契约不变性族）

### EK-M02 · 审计 = 结构化本地单行 JSON，事件名闭合枚举
- **层**: L1/L2
- **内容**: audit.rs 定义本地诊断审计记录（非 ETW）。不变量：`"event"` 恒为第一键；字段顺序=构建调用序；值仅 string/int/bool（集合用逗号连接+`_count` 伴生）；字符串经 serde_json 转义保一行一条；事件名来自闭合 `AuditEventName` 枚举（typo=编译错误）。内容规则：**无自由文本、无配置值/路径/命令行/UPN**，错误细节降为数值码；调用方 ID（containerId 等）须经 `sanitize_identity`。记录只进辅助诊断 sink（--log-file / MXC_DIAG_CONSOLE 命名管道），**永不达主 console/buffer sink（= SDK 调用方捕获输出）**。
- **证据**: src/mxc-sdk/src/core/mxc_common/audit.rs:1-70、AuditEventName 枚举
- **links**: constraint→EK-M01（PolicyHash 是审计记录之一）；subsystem↔EK-M05（同为 mxc_common 治理组件）

### EK-M03 · 引擎 = 后端选择唯一所在地（mxc_common 不含后端依赖）
- **层**: L1/L2
- **内容**: `mxc_engine` 明确声明为"把执行请求变成运行沙箱"的唯一实现点：SDK 与 JSON parser 都归一化为 `ExecutionRequest` 后进入引擎；后端选择（resolve_runner / run_state_aware / spawn_runner）只在这里发生。`mxc_common` 必须不依赖 `backends/*`（它是那些后端构建其上的跨平台基础）。streaming 分发只在 dispatch.rs（ProcessContainer/Bubblewrap/LXC/Seatbelt/WSLC/IsolationSession 有 streaming 路径；其余实验性后端返回 `unsupported_containment`，需驱动独立执行器二进制）。
- **证据**: src/mxc-sdk/src/core/mxc_engine/mod.rs:1-30（模块文档）；dispatch.rs:1-30、spawn_runner 匹配
- **links**: dependency→EK-M09（状态感知经 resolve_backend→dispatch_state_aware）；subsystem↔EK-M04

### EK-M04 · 后端注册表 = 静态 10 项（backend+experimental+probe_coverage）
- **层**: L1
- **内容**: `static BACKENDS: [BackendRegistration; 10]`——ProcessContainer/Wslc/Lxc/Vm/MicroVm/Hyperlight/WindowsSandbox/IsolationSession/Seatbelt/Bubblewrap。experimental 标记 3 个（MicroVm/Hyperlight/WindowsSandbox）。probe_coverage 区分 Detected(&[target]) / NotDetected(解释)。含防退化测试（注册键唯一、实验性集合保持、规范名=wire 名）。
- **证据**: src/mxc-sdk/src/core/mxc_engine/backend_registry.rs:1-120、tests
- **links**: dependency→EK-M08（experimental.rs 查询 registration）；mechanism↔EK-M10（Probe/平台支持同族）

### EK-M05 · 沙箱进程 = 流式句柄抽象（SandboxProcess trait）
- **层**: L1/L2
- **内容**: `SandboxProcess: Send` trait 定义统一流式句柄：warnings/output_metadata/take_stdin/take_stdout/take_stderr/try_wait/wait/kill/kill_for_timeout/pty_*。stdio 管道或 PTY 两种模式；`kill_for_timeout` 与普通 `kill` 分离（超时强杀语义独立）。`NativeStdio` 持原生句柄（duplicate_and_take_native_stdio）。库上下文调用者有生命周期契约：**dlopen 库必须比所有 spawn 句柄活得久**（ETW provider 回调残留问题，Drop 释放）。
- **证据**: src/mxc-sdk/src/core/mxc_common/sandbox_process.rs:149-336、mxc_engine/mod.rs:80-100（库生命周期契约）
- **links**: subsystem↔EK-M06；dependency→EK-M11（PTY 复用）

### EK-M06 · 引擎内 streaming vs run-to-completion 双路径
- **层**: L1/L2
- **内容**: 引擎提供 `spawn_execution_request`（streaming handle，backends 无实现则 UnsupportedContainment）与 `run`（run-to-completion，Windows ProcessContainer 走三层 fallback dispatcher）。`dry_run` 在 streaming 路径被**显式拒绝**（"validate don't execute"——没有进程可流，静默忽略是 bug）。两者都先 `log_policy_hash` 锚定策略身份。
- **证据**: dispatch.rs:37-60（dry_run 拒绝）、mxc_engine/mod.rs:106-120、run.rs:105-110（log_policy_hash）
- **links**: contrast↔EK-M09（streaming vs 生命周期）；causal→EK-M01（都发 PolicyHash）

### EK-M07 · 审计与日志分离（诊断 sink 永不达主输出）
- **层**: L1/L2
- **内容**: Logger 双模式：`Mode::Buffer`（SDK 调用方捕获）+ 辅助诊断 sink（--log-file / MXC_DIAG_CONSOLE 命名管道）。审计记录**只**走辅助 sink，从设计上隔离"诊断"与"调用方输出"——诊断日志可安全附到 bug report（无配置值/路径/凭据），调用方输出不掺审计噪音。
- **证据**: audit.rs:1-25（"deliberately never reaches the primary console/buffer sink"）；logger.rs:Mode
- **links**: constraint→EK-M02（审计记录 sink 规则）；contrast↔EK-M13（telemetry 是另一通道）

### EK-M08 · 实验性 = 调用方 opt-in，不是 JSON 字段
- **层**: L2（安全关键）
- **内容**: `require_experimental_optin(backend, experimental_enabled)`：注册表标记 experimental 的后端，若调用方未启用 experimental 则 `backend_unavailable`。**关键设计：experimental 是函数参数而非 JSON 字段**——"a configuration cannot grant itself experimental access"（配置不能自我授权）。防止攻击面扩展仅由策略文档触发。
- **证据**: src/mxc-sdk/src/core/mxc_engine/experimental.rs:1-30、mxc_engine/mod.rs:117-120（spawn_one_shot_json 注释）
- **links**: constraint→EK-M04（注册表 experimental 标记为权威）；contrast↔EK-M01（PolicyHash 也排除 experimental_enabled——授权不改变 enforcement）

### EK-M09 · 状态感知生命周期 = Provision/Start/Execute/Stop/Deprovision 五阶段
- **层**: L1/L2
- **内容**: `StateAwareOperation` 枚举（Provision(StateAwareProvision) / Start{sandbox_id} / Stop / Deprovision）+ `Phase`（Provision/Start/Execute/Stop/Deprovision）。`StatefulSandboxBackend` trait 定义各阶段 config/metadata 关联类型；`StateAwareProvision` 变体（IsolationSession/Wslc）决定 phase 与路由数据。`dispatch_state_aware` 按操作分发，`resolve_backend` 先解析后端再 `require_experimental_optin`。sandbox_id 是跨阶段 join key（SandboxIdentity 审计记录每次生命周期发一次）。
- **证据**: state_aware_operation.rs:43-120、state_aware_dispatch.rs:136-200、state_aware_backend.rs:221-280
- **links**: causal→EK-M12（学习模式在生命周期中捕获）；dependency→EK-M03（经引擎 dispatch）

### EK-M10 · 平台支持探测 = 只读 availability 探针
- **层**: L1
- **内容**: `platform_support` / `PlatformSupport`（含 BubblewrapNetworkSupport/ProxyEnforcement）+ `available_backends`（只读探针，带 effective isolation tier）+ `BackendCapability`。Windows 另有 `probe_execution_request`（ProbeFacts/ProbeOutput/UiCapabilitySupport）与 `guarded_capture_available`。探测与执行分离：探针是只读的，不建容器。
- **证据**: mxc_engine/mod.rs:40-60（pub use）、platform.rs、probe.rs
- **links**: dependency→EK-M04（probe_coverage 静态表）；mechanism↔EK-M06

### EK-M11 · PTY 支持 = spawn_with_pty / PTY 句柄复用
- **层**: L1
- **内容**: `spawn_with_pty`（merged terminal output + resize）+ `spawn_one_shot_pty_json`；SandboxProcess 提供 pty_clone_reader / pty_take_writer / pty_resize / pty_size。PTY 是普通 stdio 路径的并行通道（"ordinary run and spawn paths allocate no PTY"）。IsolationSession 有 console_relay/console_mode（Windows 独立会话的控制台中继）。
- **证据**: lib.rs:30-40、sandbox_process.rs:177-225、isolation_session/common/console_relay.rs
- **links**: subsystem↔EK-M05；mechanism↔EK-M06（双路径的 PTY 变体）

### EK-M12 · 学习模式 = 先观察后强制的策略打磨闭环
- **层**: L2
- **内容**: 学习模式核心：`DenialAnalyzer` trait（平台无关抽象）把平台原生捕获源解码为跨平台 `DeniedResource` 模型（resource + resource_type + access_type，按 (resource,accessType) 去重，首见序保留）。Windows 实现基于 sealed ETW trace（.etl）：`etl_decode`/`tdh_decode`/`etl_filter`；`process_lifetime` 用 job-observed 进程生命周期窗口过滤 host-wide 捕获（防止无关宿主事件混入）。`denied_resources_truncated` 标志位——超过 bound 截断但如实报告。三模式：Enforcement/Learning/Permissive（policy.md `captureDenials.mode: block|allow`，allow=允许并记录）。
- **证据**: learning_mode_core/analyze.rs:1-50、model.rs:41-130、learning_mode_windows/etl_decode.rs、process_lifetime.rs、docs/containment-policy/1.0.0/policy.md:135（captureDenials）
- **links**: causal→EK-M09（生命周期内捕获）；constraint→EK-M01（PolicyHash 排除 capture_denials.output_path——不改变 enforcement 不哈希）

### EK-M13 · 遥测同意 = 显式 consent，非 Windows 不采集
- **层**: L1/L2
- **内容**: `ConsentState`（Granted/Denied/Undetermined）+ `request_consent` 交互；非 Windows 主机不采集遥测（ConsentState 文档明示）。consent 是调用方选项，**不是策略字段**（policy.rs: "Telemetry is intentionally not a policy field"——调用仪器而非沙箱限制）。ETW provider 引用计数注册/释放（telemetry::init / TelemetryProcess Drop）。
- **证据**: telemetry.rs:1-60、policy.rs:625-635、mxc_engine/mod.rs:80-100
- **links**: contrast↔EK-M07；constraint→EK-M01（telemetry 被 PolicyHash 排除——无 enforcement 效果）

### EK-M14 · 网络策略 = 显式方向语义 + 后端能力拒绝
- **层**: L1/L2
- **内容**: `NetworkPolicy`（egress + ingress）+ `NetworkAction`（**默认 Deny**）+ `NetworkPeerPolicy`（CIDR）+ `NetworkRuntimeConfig`（network_proxy）。Bubblewrap 后端**显式拒绝** ingress.default='allow' / hostLoopback='allow'（slirp 私有网络命名空间无名路由进）；proxy 与直接 egress 规则互斥（proxy-only 链会静默丢弃 egress 规则 → 编译期拒绝）；`validate_network_policy_support` 在无 CIDR/端口支持的后端拒绝规则请求。
- **证据**: policy/network.rs:1-90、bwrap_command.rs:204-260（directional_network_rejection + 拒绝文本）、validator.rs:61-140
- **links**: causal→EK-M15（filesystem/UI 同类策略域）；subsystem↔EK-M15↔EK-M16（策略域三件套）

### EK-M15 · 文件系统策略 = 显式路径清单 + 全别名规范化
- **层**: L2（安全关键）
- **内容**: `FilesystemPolicy`（readonlyPaths/readwritePaths/deniedPaths）+ `AccessMode`（Read/ReadWrite）。**绕过防护核心**：`canonicalize_path` 用 `GetFinalPathNameByHandleW` 折叠 symlink/junction/8.3 短名/`\\?\` 前缀别名到唯一规范路径——"on-disk 别名只有在 OS 解析后才分叉，因此必须先用句柄问 OS 最终路径"；`canonicalize_allowing_absent_tail` 容忍未创建叶（解析最深存在祖先，防止 symlink 挂载树逃逸）。策略挂载在 `--tmpfs /tmp` 之后（用户策略优先于标准挂载）。
- **证据**: filesystem_canonical.rs:1-200、filesystem_access.rs:1-40、bwrap_command.rs:347-350、policy.rs:545-553
- **links**: constraint→EK-M01（readwritePaths 敏感于 PolicyHash）；mechanism↔EK-M16（Windows DACL/BFS 同族）

### EK-M16 · ProcessContainer 三层 fallback + DACL 恢复
- **层**: L1/L2
- **内容**: Windows ProcessContainer 按宿主能力选 Tier1 BaseContainer / Tier2 AppContainer+BFS / Tier3 AppContainer+DACL（fallback_detector::detect）。T1 无宿主 DACL（PSEC API 不应用）；T2 BFS 模型只能表达允许列表（不能选择性拒绝父路径）→ 需额外 DACL deny 增强；Tier3 的 DACL `Drop` **恢复宿主 ACE**（guard 必须活过 run——先 drop runner 再 drop guard）。`allowDaclMutation: false` 拒绝宿主 DACL fallback。
- **证据**: process_container/common/dispatcher.rs:1-50、docs/containment-policy/1.0.0/policy.md:43-46、135
- **links**: contrast↔EK-M15（Linux 挂载 vs Windows DACL 两解法）；mechanism↔EK-M03（run.rs 驱动 dispatcher）

### EK-M17 · 契约版本化 = 精确匹配 + 发布即冻结
- **层**: L2
- **内容**: `ContractVersion`（V0_9_0Alpha/V1_0_0/V1_1_0Alpha）+ `probe_version`（serde 探针取 version 字段，`parse_exact` 精确拼写匹配，**无语义版本比较**——未注册版本→`UnsupportedVersion`，非法声明→`InvalidDeclaration`）。`ContractStatus`（Published/Development）分离。schema 文件自述 `PUBLISHED IMMUTABLE FILE - DO NOT EDIT`——1.0.0 是权威不可变契约；`schema-version.json` 是版本权威源（min/maxSupported/stableLatest/stateAware/sdkMajorTargets）。
- **证据**: mxc_contract/version.rs:1-90、registry.rs、schemas/stable/mxc-config.schema.1.0.0.json:1-10、schemas/schema-version.json
- **links**: constraint→EK-M18（解析器必须精确支持 min..max）；mechanism↔EK-M20（SDK 面契约映射）

### EK-M18 · schema 支持 = min/maxSupported 精确窗口
- **层**: L1
- **内容**: `mxc_schema_support` 模块承载版本窗口语义：schema-version.json 的 `min=0.9.0-alpha` / `maxSupported=1.1.0-alpha` 定义解析器精确支持范围；`stableLatest=1.0.0`；`stateAware`/`stateAwareWindowsSandbox`/`stateAwareWslc` 独立版本化（IsolationSession=0.9.0-alpha，WindowsSandbox=1.1.0-alpha）；`sdkMajorTargets: {1: 1.0.0}` 映射 SDK 大版本到最新发布精确契约。发布版契约 is authoritative for declared 版本请求。
- **证据**: schemas/schema-version.json、mxc_contract/registry.rs
- **links**: dependency→EK-M17；subsystem↔EK-M03

### EK-M19 · 校验分层 = common 校验 + 后端能力校验 + exec 阶段校验
- **层**: L1/L2
- **内容**: `validate_common`（请求级通用校验，reject 前置到后端校验前）、`validate_network_policy_support`（后端能力 gate：CIDR/协议/端口 allow/deny 规则支持，不支持则 `ScriptResponse::rejected` 明确拒绝而非静默忽略）、`validate_exec_common`（exec 阶段、后端自身 validate_exec hook 之前）、`validate_port_mappings`（零端口/非 TCP 协议拒绝）。错误→闭合 `MxcErrorCode` 枚举（malformed_request/unsupported_containment/unsupported_phase/backend_unavailable/policy_validation…）+ envelope 包装。
- **证据**: validator.rs:1-210、mxc_error.rs:1-260
- **links**: causal→EK-M06（校验失败→拒绝执行）；subsystem↔EK-M14（网络校验是其实例）

### EK-M20 · 非 SDK 消费 = 独立执行器 + 版本化 JSON
- **层**: L1/L2
- **内容**: 非 SDK 消费路径：`wxc-exec.exe`（Windows 平台特定执行器二进制）接受 `schemas/stable/` 定义的 JSON 容器创建请求——"Use for testing or when the SDK cannot be embedded in your app"。`tools/wxc`（Rust）+ `tools/lxc` + `tools/mxc_darwin`（macOS）+ `mxc_schema_gen`（schema 生成器）+ `mxc_diagnostic_console`（诊断）。`wxc-exec.exe` 经 build.rs 嵌入版本信息。JSON 请求走同一引擎（spawn_one_shot_json / run_state_aware_json / exec_state_aware_json）。
- **证据**: README.md:60-70（Non-SDK consumption）、tools/wxc/build.rs:10、mxc_engine/mod.rs:120-130（spawn_one_shot_json）、state_aware.rs:909-940（*_json 入口）
- **links**: contrast↔EK-M03（进程内 SDK vs 独立执行器）；mechanism↔EK-M17（JSON 版本契约）

## EK Graph 子图（关键簇）
```
EK-M01(PolicyHash) ──causal──▶ EK-M02(审计) ──constraint──▶ EK-M07(sink 隔离)
     │                                                          │
     └──constraint──▶ EK-M12(学习模式: output_path 排除)   ┌──▶ EK-M13(遥测: 非策略字段)
                                                          │
EK-M03(引擎唯一) ──dependency──▶ EK-M09(生命周期) ──causal──▶ EK-M12(学习捕获)
     │subsystem
EK-M04(注册表) ──dependency──▶ EK-M08(实验性 opt-in)
     │mechanism
EK-M10(平台探测) ◀──dependency── EK-M04
EK-M14(网络) ──subsystem──▶ EK-M15(文件系统) ──constraint──▶ EK-M01
     │
EK-M16(Windows DACL) ──contrast── EK-M15(Linux 挂载)
EK-M17(契约版本) ──constraint──▶ EK-M18(schema 窗口) ──subsystem──▶ EK-M03
EK-M20(独立执行器) ──contrast── EK-M03(进程内 SDK)
```

## 质量指标（本层）
- EK 总数: 20（M01~M20）
- 全部带 links；平均出边 ~1.6；游离 0%
- 证据强度: S3~S4（实现 + 测试锁定）
