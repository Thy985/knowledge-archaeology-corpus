# Flow Atlas — MXC 七类流（从真实代码导出）

> 关键 Edge 可回溯 symbol/file/condition。

## 1. Control Flow（控制流）
```
App Launch API → mxc-sdk v1::run/spawn → mxc_engine::resolve_runner
  → [windows|linux|macos cfg 分支]
    → Backend 判定：registered? experimental? host? compiled?
      → unsupported_containment / backend_unavailable 错误路径
    → ScriptRunner 构造 → run to completion / streaming handle
state-aware 支流：provision → (start) → exec → stop → deprovision
  → state_aware_dispatch::resolve_backend（sandboxId 前缀 iso:/wslc:）
```
- **关键 Edge**: run.rs resolve_runner（"single home"）；binaries 无 selection match（copilot-instructions 不变量）

## 2. State Flow（状态流）
```
容器状态机（IsolationSession/WSLc）：provisioned → started → exec → stopped → deprovisioned
创建即执行（run-to-completion）：implicit destroyOnExit=true 默认
persistent: destroyOnExit=false + preservePolicy=false（policy 不保留跨重建？preservePolicy 独立布尔）
```
- **关键 Edge**: sandboxId 前缀（iso:/wslc:）推断后端；exec 继承 provision 网络姿态（"inherit the provisioned network posture"）；生命周期阶段字段 whitelist（policy.md phase 表）

## 3. Data Flow（数据流）
```
typed SDK request / raw MXC request JSON
  → mxc_contract registry（version 注册校验）
  → version-specific adapter（v0_9 / v1_0 / dev）
  → shared normalization（ExecutionRequest / typed lifecycle op）
  → mxc_engine routing + authorization
  → backend host-capability validation → enforcement
```
- **关键 Edge**: versioning.md convergence mermaid；"SDK policy types and generated schemas do not form another native configuration authority"；FFI 无 free-form 格式（schema.md）

## 4. Evidence Flow（证据流）
```
workload denied access（deny-by-default）
  → OS 返回 Access denied + [learning] OS 记录
  → capture（Windows sealed ETW / native PSEC captureDenials / WPR plm）
  → DenialAnalyzer（跨平台 trait）decode
  → AnalysisResult：bounded 去重 + truncated 标志 + process-lifetime 窗口过滤 + username-redacted
  → denials.json / denials.verbose.json（plm）→ compatibility adjusted-config generator
```
- **关键 Edge**: analyze.rs "preventing unrelated host [events]"（窗口过滤）；plm "host-wide ETL does not cross the privilege boundary"（权限边界在分析器侧）

## 5. Authority Flow（权威流）
```
谁是权威：immutable published contract（mxc_contract registry）
  → 引擎路由授权（mxc_engine: experimental_enabled 门 / backend availability）
  → 后端强制（host capability validation）
管理权威：HKLM AllowTelemetry（ceiling never grant——策略可阻止不可授权）
特权边界：elevated guardian（plm）分析于 protected scratch；host-wide 源 ETL 不跨边界
```
- **关键 Edge**: versioning.md 权威表四行；telemetry.md "ceiling, never a grant"；plm readme

## 6. Memory Flow（记忆流）
```
持久化容器：state-aware lifecycle（provision 返回 sandboxId 供后续 phase 引用）
session 关联：wslc.provision.sessionId（新沙箱加入既有 session）
exec 网络姿态记忆：exec 继承 provision posture（reject supplied policy at exec）
```
- **关键 Edge**: policy.md lifecycle 段 + provision/exec 示例；state-aware-wslc-* 测试

## 7. Policy Flow（策略流）
```
开发者声明（containment policy 三域 FS/Network/UI）
  → 版本化契约（closed JSON 校验）
  → 引擎路由（后端匹配策略 section 拒绝不支持的）
  → 后端强制（backend-specific enforcement；Seatbelt profileOverride 例外）
  → Learning 反馈：deny-and-record（不变）→ DenialAnalyzer → 最小权限候选 → 修订声明
```
- **关键 Edge**: 三模式（Enforcement/Learning/Permissive——logging-access-denied.md 双 capability）；"Unsupported…policies are rejected…not silently applied"；policy.rs 策略发现（available_tools_policy 等枚举宿主）

## Flow 真实性声明
- 每条 Edge 均可回溯文件锚（本文件 + EK Graph 证据列）；无凭记忆 Flow
