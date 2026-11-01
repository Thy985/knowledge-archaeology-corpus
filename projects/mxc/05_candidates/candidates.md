# 05 · Candidates — MXC 未验证假设（严格区分 Hypothesis/Observation）

> 不能确认的内容留在候选，不冒充知识。每条含证据缺口与验证路径。

## C-M01 · 双层策略（workload + 组织 Intune 叠加）的强制优先级（Hypothesis）
- **假设**: 组织策略（Intune 管理面）叠加/覆盖 workload 声明，且强制优先级由管理面决定
- **当前证据（Observation）**: README 明示 "policies enforced by the developer **or organization**"；`docs/containment-policy/README.md` 提到 workload + organization 两层
- **缺失证据**: 本次考古未见 Intune 策略下发/合并/优先级冲突的**代码实现**（引擎侧仅有单层 workload 策略消费路径）；未能定位策略合并模块
- **验证路径**: 检索 host/ 目录（wxc_host_prep）、docs/ 中 organization policy 下发文档；或 Intune MXC policy CSP 定义
- **Epistemic**: Hypothesis（强烈指示但无引擎证据）

## C-M02 · 学习模式三态（Enforcement/Learning/Permissive）的运行时状态机（Observation→Hypothesis）
- **Observation**: `captureDenials.mode: block|allow` 已实现（policy.md:135）；DenialAnalyzer/ETW 解码已实现
- **Hypothesis**: "Learning（阻断+记录）与 Permissive（允许+记录）是两个独立运行时姿态"——源码中 learning_mode 是捕获+分析组件，未看到运行时"允许一切但记录"的独立执行姿态枚举
- **验证路径**: 检索 wxc_test_driver / mxc_diagnostic_console 对 mode 的消费；或 README 三运行模式文档对照 engine 代码

## C-M03 · "agent 不能成为自身安全权威"是产品层原则而非引擎层机制（Observation）
- **Observation**: 该句出现在 README 产品定位，引擎层未发现自证性机制（如禁止 agent 读取自身策略的代码 gate）
- **推断**: 这是**架构立场**（策略由宿主进程传入，agent 进程内不可改），而非运行时防御
- **验证路径**: 对照 dsh（sandbox/approval seam）与 AGT（无状态引擎）——MXC 的强制点在 OS 边界

## C-M04 · MicroVM/WindowsSandbox 学习模式尚未实现（Observation）
- **Observation**: 注册表标 MicroVm/WindowsSandbox 为 experimental 且 probe_coverage 为 NotDetected（"no hypervisor-presence detector yet"）；`denials` 学习路径在 Windows ETW + Bubblewrap/LXC 语境
- **缺失证据**: 未发现 MicroVM 后端的 DenialAnalyzer 实现
- **验证路径**: 检索 backends/nanvix-microvm 的 capture 支持

## C-M05 · 遥测 ETW provider 的跨进程安全边界（Hypothesis）
- **假设**: MXC 的 ETW telemetry（Windows）注册于调用进程，provider 回调生命周期与库加载绑定
- **当前证据（Observation）**: mxc_engine/mod.rs 明确 dlopen 生命周期契约（provider 引用计数 + Drop 释放）
- **缺失证据**: 若调用方崩溃/强杀，provider 是否泄漏注册（非本仓库可验证——需运行观察）
- **验证路径**: Windows 宿主集成测试

## C-M06 · 跨项目假说：执行边界指纹化（Cross-project Candidate）
- **假说**: 沙箱系统的"每次运行的策略指纹"（MXC PolicyHash）与 RAMPART 的"不可变版本化 schema 决策门"是同一治理模式的两种实现——**可审计执行 = 身份 + 契约 + 证据三件套**
- **当前证据**: MXC（PolicyHash + 版本化 JSON 契约 + 审计事件）vs RAMPART（trace schema 兼容性门 + 版本化决策）——单项目各一
- **验证路径**: AGT（无状态策略引擎）第三方对照；若 AGT 也有 request fingerprinting，则升为 Pattern
- **Epistemic**: Cross-project Candidate（不是已验证 Principle）

## C-M07 · 路径规范化跨平台统一（Observation→Hypothesis）
- **Observation**: Windows 有全别名 canonicalization（GetFinalPathNameByHandleW）；Bubblewrap 有 resolve_through_symlinks（canonicalize_allowing_absent_tail 注释明确镜像它）
- **Hypothesis**: "所有后端共享同一路径规范化不变量"——当前是平台各自实现，抽象层（PathCanonical 三态）是共享模型
- **验证路径**: LXC/Seatbelt 后端的路径处理对照（本次未逐一深读）
