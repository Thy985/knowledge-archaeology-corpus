# Project Layer — MXC 项目地图（L0 事实）

> 全部事实可追溯仓库实际内容（commit ffecec0）

## 是什么
沙箱化代码执行系统：SDK 依赖嵌入应用（in-process），应用指定容器类型 + 遏制规则 + workload 命令，MXC 验证请求、选后端、启动隔离 workload。（README.md:29-43）

## 架构（三层 + 引擎）
```
App (Launch API) → SDK (Rust/.NET/Node, in-process) → Backend (selected, in-process) → Isolated workload
```
- **mxc-sdk**：公开 API 层（v1::ContainerRequest / run / spawn / spawn_with_pty / state-aware lifecycle）
- **core/mxc_common**：跨平台 foundation（models/contract adapters/engine/learning_mode/telemetry）
- **core/mxc_engine**：唯一执行引擎（backend_registry / dispatch / resolve_runner / state_aware / experimental / probe）
- **core/mxc_contract**：版本化契约注册（published immutable / development mutable / registry.rs ContractStatus）
- **backends/**：9 后端实现（bubblewrap/hyperlight/isolation_session/lxc/nanvix/process_container/seatbelt/windows_sandbox/wslc）
- **tools/**：wxc-exec（Windows CLI）/ lxc-exec / mxc-exec-mac / mxc_diagnostic_console / mxc_schema_gen / plm（Learning Mode）/ mxc_darwin
- **ffi/mxc_ffi**：C-callable 库（同一契约，无 free-form 格式）

## 核心数据结构
- MXC request JSON（版本化 closed 对象）：version/process/containment/filesystem/network/ui/runtimeConfig/lifecycle/telemetry
- 策略三域：filesystem（readwritePaths/readonlyPaths/deniedPaths）、network（egress/ingress 方向性 + hostLoopback + proxy）、ui（disable/clipboard/injection）
- 后端注册表：BACKENDS[10]（ContainmentBackend + experimental + ProbeCoverage）
- AnalysisResult（learning mode）：denials（bounded 去重）+ denied_resources_truncated + verbose_logging（username-redacted）
- 生命周期 ID：sandboxId（iso:/wslc: 前缀）+ sessionId（wslc.provision.sessionId）

## 状态与生命周期
- run-to-completion（创建即执行完）/ streaming（spawn 句柄）/ state-aware（provision→start→exec→stop→deprovision，仅 IsolationSession+WSLc）
- lifecycle.destroyOnExit（默认 true）/ preservePolicy（默认 false）

## 测试体系
- tests/regression_tests：PowerShell per-Issue 回归（TestCases.psd1 目录，非破坏性仅限，wxc-exec 依赖）
- tests/policy：sdk-v1（input/expected/invalid 三集）——契约一致性金标准（含 unknown-field 无效集）
- tests/fixtures + configs；src/testing：fuzz（OneFuzzConfig）/ unix_test_proxy / wxc_test_driver / wxc_test_proxy / wxc_ui_probe
- 引擎不变量测试：probe-coverage invariant tests（backend_registry 读注释）

## 配置与版本治理
- schemas/stable：1.0.0（immutable published）+ 0.4~0.9 alpha（历史演进）；schemas/dev：1.1.0-alpha（可变）
- schemas/schema-version.json：权威源（min 0.9.0-alpha / maxSupported 1.1.0-alpha / stableLatest 1.0.0 / sdkMajorTargets {1: "1.0.0"}）
- containment-policy docs 按版本（0.9.0/1.0.0/dev）

## 权限与治理机制
- 策略契约 immutable；未支持的策略**拒绝**而非静默应用（copilot-instructions 架构不变量）
- 权限边界：elevated guardian 分析于 protected scratch；host-wide ETL 不跨权限边界（plm readme）
- 管理策略：telemetry HKLM\SOFTWARE\Policies\Mxc AllowTelemetry（ceiling never grant，未知值 blocked）
- agent 协作治理：AGENTS.md + .github/copilot-instructions.md + .github/skills/mxc-code-review（review skill）

## 外部依赖
- workspace.dependencies：anyhow/base64/chrono/cidr/clap/flatbuffers 等（src/Cargo.toml）
- 平台系统依赖：PSEC（ProcessContainer）/ Seatbelt / Bubblewrap+seccomp / LXC / WPR ETW（learning mode）/ wslcsdk.dll（wslc feature 门控）
- 构建：rust-toolchain.toml 固定；build.sh/build-mac.sh/build.bat；node/dotnet SDK 带 native 资产
