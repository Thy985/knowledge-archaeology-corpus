# 04 · Flow Atlas — MXC 七类流（HEAD fda29ac）

> 每条关键 Edge 可回溯 symbol/file/condition/state transition。

## 1. Control Flow（执行控制）
```
v1::run / v1::spawn / v1::spawn_with_pty（lib.rs 公共入口）
  → ContainerRequest::into_sdk_input（policy.rs → sdk_input.rs）
  → run_typed_state_aware_request / spawn_execution_request（engine/mod.rs）
  → resolve_runner / spawn_runner（run.rs / dispatch.rs）
      ├─ ensure_host_supported()（非 Win/Linux/macOS → UnsupportedContainment）
      ├─ log_policy_hash(request)（每次运行锚定策略身份）
      └─ match containment → 具体后端 runner
  → SandboxProcess 句柄（stdio 管道 或 PTY）
  → wait / kill / kill_for_timeout（sandbox_process.rs）
```
- Edge: `ContainerRequest → ExecutionRequest` 归一化是**唯一**入口（SDK 与 JSON parser 都经此）；`dry_run` 在 streaming 被显式拒绝（dispatch.rs:40-45）
- State transition: one-shot（run/spawn）vs state-aware（Provision→Start→Execute→Stop→Deprovision，state_aware_operation.rs:43-120）

## 2. State Flow（状态）
```
StateAwareOperation（Provision/Start/Stop/Deprovision）→ Phase 映射（operation→phase）
  → StatefulSandboxBackend::provision → ProvisionResult{sandbox_id}
  → Start{sandbox_id} → Execute（返回 streaming handle）
  → Stop{sandbox_id} → Deprovision{sandbox_id}
sandbox_id = 跨阶段 join key（SandboxIdentity 审计记录每次生命周期一次）
```
- Edge: `StateAwareProvision` 变体决定 phase 与路由数据（IsolationSession/Wslc 配置）
- 状态错误：NotProvisioned/NotStarted/AlreadyStarted/AlreadyStopped/StaleId 闭合错误码（mxc_error.rs:230-241）

## 3. Data Flow（数据）
```
SDK 类型（ContainerRequest）→ into_sdk_input → SdkStateAwareInput → ExecutionRequest（引擎内部规范化）
JSON 路径：spawn_one_shot_json / run_state_aware_json（原始 JSON → probe_version → 解析器 → ExecutionRequest）
  → 后端 runner（Bubblewrap: build_args → bwrap 命令参数数组）
  → 进程 stdio 管道 / PTY
  → ScriptResponse / ExecOutcome（stdout/stderr/exit/timeout）
```
- Edge: `probe_version` 精确匹配版本→解析（mxc_contract/version.rs:79-83）；未知版本→UnsupportedVersion
- 文件系统数据：canonicalize_path → PathCanonical（Canonical/Unknown/Absent 三态）——路径在进策略前规范化

## 4. Evidence Flow（证据）
```
每次运行 → log_policy_hash → mxc.PolicyHash 审计记录（策略指纹）
后端执行 → 审计事件（mxc.ProcessExited/ProcessTimedOut/EnforcementDegraded/ConfigRejected…）
  → Logger::log_audit_event → 仅辅助诊断 sink（--log-file / MXC_DIAG_CONSOLE）
  → 永不达主 console/buffer sink（audit.rs:1-25）
学习模式：捕获源（Windows ETW .etl / 未来 Linux）→ DenialAnalyzer → DeniedResource 模型
  → process_lifetime 窗口过滤（job-observed PID+timestamp 防止无关宿主事件混入）
```
- Edge: `denied_resources_truncated=true` 如实报告截断（analyze.rs:30-35）——不静默丢证据

## 5. Authority Flow（权威）
```
策略权威 = 开发者 workload 声明 + 组织 Intune 管理策略叠加（双层策略，docs/containment-policy）
  → JSON 契约（版本化，发布即冻结）
  → 引擎验证（validate_common → validate_network_policy_support → validate_exec_common）
  → 后端强制（OS 原语：Bubblewrap 挂载/AppContainer/DACL/Seatbelt）
  → 实验性后端 gate（require_experimental_optin：调用方 opt-in，非 JSON 字段）
```
- Edge: 权威链的**关键不变量**——"agent 不能成为自身安全权威"：策略在 workload 控制之外由 OS 独立强制；配置不能自我授权（experimental/telemetry 都是调用方参数）

## 6. Memory Flow（记忆/持久状态）
```
沙箱生命周期状态：sandbox_id → backend 内部对象（Job object / AppContainer profile / WSL distro）
  → 状态文件（DACL tier 的 state-file）
  → 生命周期操作显式管理（provision/start/stop/deprovision）
  → DestroyOnExit 默认 true（生命周期策略 lifecycle.destroyOnExit，policy.md:43）
无跨运行持久记忆：一次运行 = 独立容器；PreservePolicy 默认 false
```
- Edge: `preservePolicy: false` 默认——策略不留存（与学习模式 activity report 形成对照：报告可留存、策略本体默认不保留）

## 7. Policy Flow（策略/治理闭环）
```
策略声明（JSON/SDK 类型，省略=默认拒绝）
  → schema 版本校验（probe_version 精确匹配 + min/maxSupported 窗口）
  → 后端能力校验（validate_network_policy_support：不支持则 rejected 而非忽略）
  → 执行（Bubblewrap: --unshare-net / --die-with-parent / --unshare-user…）
  → 审计（PolicyHash + 事件记录）
  → 学习模式反馈（DeniedResource → 起草最小权限策略 → Learning→Permissive→Enforcement 渐进）
```
- Edge: `NetworkAction::default() = Deny`（network.rs:24-30）——默认拒绝是策略语言语法，不是运行时兜底
- Edge: Bubblewrap `--die-with-parent`（bwrap_command.rs:437）——bwrap 句柄被杀时容器随之销毁（无孤儿沙箱）
