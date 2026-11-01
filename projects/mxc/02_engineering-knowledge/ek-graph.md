# Engineering Knowledge Graph — MXC（L1/L2 宽底座，20 条）

> 每条含机制/证据/links；EK 是图的节点不是孤立条目。证据锚 = 仓库文件:行。
> 缩写：plm=src/host/plm；core=src/mxc-sdk/src/core；bw=bubblewrap

## EK-M01 · 契约收敛流（typed SDK 与 raw JSON 汇入唯一契约）
- **机制**: typed SDK policy/lifecycle 请求与 "MXC request JSON" 都经 contract registry → version-specific adapter → shared normalization → engine routing → backend；SDK policy types 与 generated schemas **不构成另一套 native 配置权威**
- **证据**: docs/development/architecture/versioning.md（convergence mermaid + "SDK policy types and generated schemas do not form another native configuration authority"）；core/mxc_common/config_contract_adapters/{v0_9,v1_0,dev}
- **links**: mechanism↔EK-M02（都是契约权威单一性）；dependency→EK-M14（三轴分离前置）

## EK-M02 · closed JSON 请求契约
- **机制**: 请求是 closed JSON object——unknown field / 所选 phase 不支持的字段一律拒绝（测试断言 "unknown field `experimental`"）；`$schema`/`_comment` 顶层例外
- **证据**: docs/containment-policy/1.0.0/policy.md（"unknown fields and unsupported fields for the selected phase are rejected"）；core/mxc_common/config_parser.rs:2013,2533-2543,2587,3819（unknown-field 测试锚）
- **links**: mechanism↔EK-M01（契约权威）；causal→EK-M18（拒绝→fail closed）

## EK-M03 · 网络策略默认 deny + deny wins
- **机制**: network.egress.default / ingress.default / ingress.hostLoopback 各自默认 "deny"（省略即 deny）；deny 优先于 allow（契约 D4）；mapped deny 规则防 fail-open（IPv4-mapped IPv6 被吞时补规则）
- **证据**: docs/containment-policy/1.0.0/policy.md（"Omitted…resolves to egress deny" + "deny wins over allow"）；backends/bubblewrap/common/network_rules.rs:678-679（"Denies precede allows so an explicit deny overrides a broader allow"）,732-737,756-757；:179（"mapped deny rule would otherwise fail open"）
- **links**: causal→EK-M04（FS 同一精神）；dependency→EK-M10（proxy 语境下默认 deny）

## EK-M04 · filesystem fail-closed 触发
- **机制**: deniedPaths 存在时，若路径解析为 Denied（权限拒/unreachable mount/I/O error——而非 cleanly missing），MXC 无法证明该路径不是不可检测别名（bind mount alias）→ **fail-closed 按 denied 处理**
- **证据**: core/mxc_common/filesystem_object.rs:13-14（"/data" bind mount alias 逃逸示例）,25-26,47,56,93-94（"the fail-closed trigger when deniedPaths are present"）,236
- **links**: mechanism↔EK-M03（默认拒绝族）；constraint→EK-M18（后端不可强制即拒绝）

## EK-M05 · 后端注册表 + ProbeCoverage 强制声明
- **机制**: 10 后端静态注册（BackendRegistration：backend/experimental/probe_coverage）；ProbeCoverage 声明是强制的——"a new backend cannot reach the enum without someone deciding whether the host probe reports it"；probe-coverage 由不变量测试读取
- **证据**: core/mxc_engine/backend_registry.rs:1-80（BACKENDS[10] + ProbeCoverage::Detected/NotDetected + 注释）
- **links**: subsystem↔EK-M06/EK-M07（引擎路由族）；mechanism↔EK-M14（注册表=契约权威的引擎侧）

## EK-M06 · experimental 门
- **机制**: experimental 后端（MicroVm/Hyperlight/WindowsSandbox）需 request.experimental_enabled，否则在任何主机解析前返回 backend_unavailable
- **证据**: core/mxc_engine/run.rs（"Experimental backends…require request.experimental_enabled"）；backend_registry.rs（experimental: true 标记）
- **links**: constraint→EK-M05（注册表判定）；contrast↔EK-M14（开发契约可变 vs 实验授权）

## EK-M07 · 后端选择单一权威（resolve_runner）
- **机制**: run.rs 是 "single home" for backend selection（把此前散在 wxc-exec/lxc-exec/mxc-exec-mac 的 match 收敛进引擎）；binaries 保持 thin；不支持/未编译 → unsupported_containment 错误
- **证据**: core/mxc_engine/run.rs:1-60（"This is the single home for…" + resolve_runner）；.github/copilot-instructions.md（"Keep wxc, lxc, and mxc_darwin thin. Do not add backend-selection matches to the binaries"）
- **links**: subsystem↔EK-M05/EK-M06；mechanism↔EK-M01（收敛哲学同源）

## EK-M08 · 双 Learning Mode capability 语义分离
- **机制**: learningModeLogging=deny-and-record（日志失败访问但**仍拒绝**，containment 不变）；permissiveLearningMode=audit/allow-all（记录每个访问并允许，**显式弱化** deny-by-default）；"must not be conflated"
- **证据**: docs/logging-access-denied.md（两 capability 表："semantically distinct and must not be conflated" + "Weakened" 标注）；policy.md（processContainer.captureDenials.mode: allow "permits and records otherwise denied access"）
- **links**: contrast↔EK-M09（观察→分析的接力）；constraint→EK-M01（能力是 policy 输入非运行时开关）

## EK-M09 · DenialAnalyzer 抽象 + 有界去重 + 生命周期窗口
- **机制**: DenialAnalyzer trait 跨平台（Windows sealed ETW；未来 Linux 自实现）；AnalysisResult 有界去重（first-seen order + denied_resources_truncated 标志）；WPR host-wide capture 只接受 job-observed PID+timestamp 窗口内事件（防无关主机事件混入）；username-redacted verbose
- **证据**: core/learning_mode_core/analyze.rs（AnalysisResult + "Inclusive process-lifetime window…preventing unrelated host [events]"）；core/learning_mode_core/{model,summary,verbose_logging}.rs
- **links**: causal→EK-M08（capture 产生 → 分析）；subsystem↔plm（src/host/plm 消费同一 analyzer）

## EK-M10 · 代理（proxy）约束与限制
- **机制**: runtimeConfig.networkProxy 仅 localhost（WSLc 例外）；代理语境下 egress.default 必须 deny 且无 rules（代理负责目标过滤）；"cooperative proxy setting alone is not a guarantee that arbitrary direct sockets are blocked"；ProcessContainer 代理需 ingress allow；identity-scoped loopback proxy 用 allowedProxyPeer
- **证据**: docs/containment-policy/1.0.0/policy.md:100-130（完整约束段）；tests/policy/sdk-v1/processcontainer-network-proxy.json
- **links**: constraint→EK-M03（代理=egress deny 的特例）；causal→EK-M11（能力差异由此而生）

## EK-M11 · IsolationSession 网络能力诚实
- **机制**: IsolationSession **无法限制网络**——请求必须显式三全 allow 且无 rules/proxy；空或缺 network 被拒绝；"Unsupported…policies are rejected by the selected backend, not silently applied"
- **证据**: docs/containment-policy/1.0.0/policy.md（IsolationSession 段落 + provision 示例）；tests/policy/sdk-v1/invalid/isolation-session-without-network.json（无效集）
- **links**: contrast↔EK-M16（WSLc 双态）；mechanism↔EK-M04（拒绝非静默）

## EK-M12 · Seatbelt profileOverride 逃生门
- **机制**: profileOverride 替换生成 profile——此时请求中 filesystem/network/ui 限制**不并入**（用户自担）；显式逃生门而非隐藏参数
- **证据**: docs/containment-policy/1.0.0/policy.md（"profileOverride replaces the generated profile, so filesystem, network, and ui restrictions…are not incorporated"）；backends/seatbelt/common/profile_builder.rs
- **links**: contrast↔EK-M11（逃生 vs 诚实受限）；constraint→KO-M01 反例

## EK-M13 · telemetry 管理策略 ceiling-never-grant
- **机制**: HKLM\SOFTWARE\Policies\Mxc AllowTelemetry（REG_DWORD，机器级，不认 HKCU）；值镜像 Windows 诊断数据刻度；未知值 **Blocked（fail closed）**；"ceiling, never a grant"——策略只能阻止不能授权，用户同意 + 管理放行双条件
- **证据**: docs/telemetry.md（表格 + "Why policy cannot substitute for consent"）
- **links**: mechanism↔EK-M03（fail-closed 族）；constraint→EK-M01（telemetry 是请求字段但受管理策略约束）

## EK-M14 · 契约三轴分离
- **机制**: versioning.md 权威表——契约存在性（mxc_contract 注册）≠ 契约发射（sdkMajorTargets）≠ 运行时授权（engine）+ 主机强制（backend）；"opening 1.1.0-alpha does not change V1's published target"；experimental 授权不使字段在旧契约合法
- **证据**: docs/development/architecture/versioning.md（决策权威表 + 三轴）；schemas/schema-version.json（sdkMajorTargets {1: "1.0.0"}）
- **links**: mechanism↔EK-M05（注册表）；constraint→EK-M02（closed JSON 依版本）

## EK-M15 · 生命周期 ID 前缀路由 + state-aware
- **机制**: sandboxId 前缀推断后端（iso:/wslc:）；provision/start/exec/stop/deprovision 仅 IsolationSession+WSLc；exec 继承 provision 网络姿态（拒绝 exec 时 supplied policy）
- **证据**: docs/containment-policy/1.0.0/policy.md（lifecycle 段落 + "infer the backend from its prefix"）；core/mxc_common/state_aware_*；tests/policy/state-aware-wslc-*
- **links**: causal→EK-M16（WSLc 双态与 lifecycle 交织）

## EK-M16 · WSLc 网络双态
- **机制**: WSLc 仅 all-deny isolated 或 all-allow bridged（egress/ingress/hostLoopback 三值一致）；拒绝混合姿态与 per-destination 规则；provision port mappings 1.0 不可用
- **证据**: docs/containment-policy/1.0.0/policy.md（"WSLc networking supports only all-deny isolated or all-allow bridged"）
- **links**: contrast↔EK-M11；subsystem↔EK-M15

## EK-M17 · 测试文化代码化（AGENTS.md）
- **机制**: 功能须有 functional validation 测试并**接入 CI**（backend 行为须在对应主机 scheduled/nightly 验证）；bug fix 须回归场景 fail-before/pass-after；"Do not judge coverage by test count"; "do not invent evidence"
- **证据**: AGENTS.md（Require meaningful behavior coverage 段落）；docs/development/build-and-test/ci-validation-infrastructure.md（validation runbook）
- **links**: mechanism↔EK-M18（治理代码化）；causal→regression_tests（tests/regression_tests 的存在即此文化的产物）

## EK-M18 · 架构不变量（unsupported policy must fail closed）
- **机制**: copilot-instructions 六不变量：mxc_common 无后端实现；mxc_engine 唯一引擎；binaries thin；build-time staging 隔离；cfg(target_os) 门控；**"Unsupported policy must fail closed. Do not accept a field that the selected backend cannot enforce."**
- **证据**: .github/copilot-instructions.md（Architecture invariants 段）；run.rs/backend_registry 与之互证
- **links**: dependency→EK-M03/EK-M04（不变量是机制的上位约束）；mechanism↔EK-M17

## EK-M19 · 策略发现（宿主枚举生成 FS 片段）
- **机制**: available_tools_policy / user_profile_policy / temporary_files_policy 枚举宿主工具/SDK/配置/临时目录生成 filesystem 策略片段（readonly/readwrite）；ContainerPolicy→build_request 映射到 ExecutionRequest
- **证据**: core/../policy.rs（"enumerate the host environment to discover tool/SDK/profile/temp directories as filesystem-policy fragments"）
- **links**: causal→EK-M01（片段并入请求）；contrast↔EK-M08（自动发现 vs 手工授权）

## EK-M20 · SDK 三语言单一契约 + FFI 无自由格式
- **机制**: Executors 与 mxc_ffi 使用同一 registered contracts；"There is no additional free-form FFI format"；Node/.NET 内部产生 request JSON；typed SDK（ContainerRequest/ProvisionRequest/ExecutionRequest）
- **证据**: docs/schema.md（"Executors and mxc_ffi…use the same registered contracts"）；README.md（SDK 表）；src/ffi/mxc_ffi
- **links**: mechanism↔EK-M01（契约收敛）；dependency→EK-M14

## EK 图统计
- EK 20 条；links 覆盖率 100%；avg 出边 ~1.9；游离 0
- 反退化检查：每条非"模块说明"（均有机制+证据+≥1 边）；无"同子系统=聚合理由"（聚合规则见 03）
