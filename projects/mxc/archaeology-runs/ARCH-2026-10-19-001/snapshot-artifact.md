# Snapshot Artifact — ARCH-2026-10-19-001（MXC · initial）

## 仓库快照
| 项 | 值 |
|---|---|
| repository | https://github.com/microsoft/mxc.git |
| HEAD commit | `fda29ac63bc306b7f97f3b86b4b63d44f7f40cf2` |
| branch | main |
| commit 主题 | "Update SDK packages to 1.0.1 (#1518)" |
| 快照时间 | 2026-10-19（UTC+8） |
| 文件数 | 2315 |
| 版本 | workspace 1.0.1（SDK 包 1.0.1） |
| license | MIT |

## 项目基础地图
- **语言**：Rust（主，~9.3MB）+ TypeScript/C#/PowerShell/Shell（SDK 包装与构建）
- **定位**：Microsoft eXecution Container（MXC）——策略驱动的跨平台沙箱代码执行系统（Windows/Linux/macOS），面向不可信代码（模型输出/插件/工具）
- **入口**：
  - SDK 库入口：`src/mxc-sdk/src/lib.rs`（`v1::run` / `v1::spawn` / `v1::spawn_with_pty`）
  - 非 SDK 执行器：`src/tools/wxc`（`wxc-exec.exe` 接受 JSON 容器创建请求）
  - 诊断控制台：`src/tools/mxc_diagnostic_console`
- **核心模块（src/mxc-sdk/src/）**：
  - `core/mxc_engine` —— 引擎（typed state-aware request 执行）
  - `core/mxc_common` —— 公共模型/沙箱进程/streaming handle（sandbox.rs 依赖）
  - `core/mxc_contract` / `mxc_schema_support` —— 契约与 schema 支持
  - `core/learning_mode_core` / `learning_mode_windows` —— 学习模式（Learning 运行模式）
  - `core/mxc_pty` —— PTY 支持
  - `policy/` —— network.rs / exact / sdk_v1_conformance.rs（策略与 V1 一致性）
  - `configs/` —— lxc.rs / process_container.rs / seatbelt.rs / wslc.rs（后端配置）
  - `backends/` —— bubblewrap / lxc / seatbelt（macOS）/ process_container（Windows AppContainer/BaseContainer）/ isolation_session / hyperlight / nanvix（MicroVM）
  - `state_aware_sdk.rs` —— 有状态生命周期（provision/start/execute/stop/deprovision）
  - `options.rs` / `sandbox.rs` / `telemetry.rs`
- **核心数据结构**：
  - `ContainerRequest`（v1）—— command + restrictions（网络/文件系统/UI 策略）+ timeout
  - `Containment` 枚举 —— Process / Bubblewrap / Lxc / Seatbelt / ProcessContainer / Wslc / IsolationSession
  - 版本化 JSON 配置 schema：`schemas/stable/mxc-config.schema.1.0.0.json`（稳定）+ 0.4~0.9 alpha 历史
  - `mxc-config.schema.1.1.0-alpha.json`（dev）
- **状态**：GA（1.0.1，2026-10-07 公告 GA）
- **测试体系**：
  - `src/mxc-sdk/src/api_tests.rs`（SDK API 测试）
  - `tests/` 顶层：configs / examples / fixtures / playground / policy / regression_tests / scripts
  - `src/testing/`：wxc_test_driver / wxc_test_proxy / unix_test_proxy / wxc_ui_probe / fuzz
- **配置与治理机制**：
  - JSON 策略 schema 版本化（schemas/stable + dev）
  - 三运行模式：Enforcement / Learning / Permissive（learning_mode 模块）
  - 双层策略（workload 声明 + 组织 Intune 管理策略）——docs/containment-policy
  - 遥测：docs/telemetry.md + core/mxc_telemetry
  - SECURITY.md / SUPPORT.md / CONTRIBUTING.md（治理文档）
- **外部依赖**：Cargo 依赖（crates.io mxc-sdk）；Node/.NET SDK 包（@microsoft/mxc-sdk / Microsoft.Mxc.Sdk）；平台原生后端（Bubblewrap/LXC/AppContainer/Seatbelt）

## 事实来源
以上全部条目来自 HEAD `fda29ac` 仓库实际内容（README.md / src/Cargo.toml / src/mxc-sdk/src/* / schemas/ / docs/）。语言构成来自 GitHub API languages 接口（Rust 9,320,281 bytes 等）。
