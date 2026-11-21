# Repository Snapshot Artifact — ARCH-2026-10-24-001（MXC）

- **project**: mxc（Microsoft eXecution Container）
- **repository**: https://github.com/microsoft/mxc.git
- **commit SHA**: `ffecec04357747d6649714f473436dfb77111be8`（main, "Add a relay to enable a loopback proxy in LXC (#1554)"）
- **version**: 1.0.1（src/Cargo.toml [workspace.package]）
- **branch**: main
- **timestamp**: 2026-10-24 02:15（Asia/Shanghai 本地）；仓库 last pushed 2026-10-23T17:54:33Z（GitHub API）
- **license**: MIT
- **规模**: Rust 246,157 行 / 513 文件；测试 Rust 33,022 行；2299 files 克隆

## 项目基础地图（均可追溯仓库实际内容）

| 维度 | 事实 | 证据 |
|---|---|---|
| 定位 | 沙箱化代码执行系统（运行不可信代码：模型输出/插件/工具），跨 Win/Linux/macOS，统一 containment model + 类型化 SDK | README.md:1-8 |
| 语言 | Rust 核心 + workspace（12 members）；C FFI（mxc_ffi）；.NET SDK + Node SDK（sdk/dotnet, sdk/node） | src/Cargo.toml [workspace] members；sdk/ |
| 入口 | mxc-sdk（Rust crate，嵌入消费应用）；wxc（tools/wxc，CLI + wxc-exec.exe 接受 JSON）；mxc_ffi（C ABI） | README.md "Non-SDK consumption"；src/tools/wxc |
| 核心模块 | host/plm（Permissive Learning Mode 引擎，Windows-only WPR helper）；backends（8 后端：ProcessContainer/Windows Sandbox/LXC/Bubblewrap/Seatbelt/MicroVM(Nanvix)/Hyperlight/IsolationSession/WSLC）；schemas（版本化 JSON）；tools（lxc/mxc_darwin/mxc_diagnostic_console/mxc_schema_gen/wxc） | src/host/plm/readme.md；README.md backends 表；docs/backends/ |
| 核心数据结构 | JSON container-creation requests + security policies（版本化 schema：stable 1.0.0 + 0.4~0.9 alpha 六版） | schemas/stable/；schemas/schema-version.json |
| 状态/生命周期 | Provision → Start → Execute → Stop → Deprovision（persistent containers）；state-aware 配置 | README.md "Container lifecycle operations"；docs/container-lifecycle.md；tests/policy/state-aware-wslc-* |
| 测试体系 | tests/regression_tests（PowerShell TestCases.psd1 + Run-RegressionTests.ps1 + test_cases/inc）；tests/policy（sdk-v1, state-aware）；tests/fixtures+configs；src/testing（fuzz / unix_test_proxy / wxc_test_driver / wxc_test_proxy / wxc_ui_probe） | tests/；src/testing/ |
| 配置 | 版本化 JSON schema（mxc-config.schema.1.0.0.json stable；alpha 演进 0.4→0.9）；containment policy（FS RO/RW/deny path lists / Network proxy+outbound+host filtering / UI clipboard+display+GUI） | schemas/stable/；docs/containment-policy/1.0.0/policy.md |
| 权限与治理机制 | 策略域三分（filesystem/network/ui）；Learning/Enforcement/Permissive 三模式（plm + native captureDenials + wxc-exec --audit）；权限边界（elevated guardian 分析于 protected scratch，host-wide ETL 不跨边界）；AGENTS.md agent 协作规则（复用正常路径/最简设计/垂直切片/文档描述成品） | AGENTS.md；src/host/plm/readme.md；docs/containment-policy/ |
| 外部依赖 | workspace.dependencies（anyhow/base64/chrono/cidr/clap/flatbuffers 等）+ 各后端系统层（AppContainer/Seatbelt/Bubblewrap/LXC/Nanvix/Hyperlight）+ .NET/Node 包资产 | src/Cargo.toml [workspace.dependencies]；sdk/ |

## 快照事实备注
- 稳定 schema 为 1.0.0（六版 alpha 并存于 schemas/stable——版本化演进可见）
- 默认后端：Win11→processcontainer；Linux→bubblewrap；macOS→seatbelt；windows_sandbox/wslc/microvm/hyperlight/isolation_session 标 experimental（README backends 表）
- docs/backends/ 含 8 后端独立文档（bwrap/hyperlight/lxc/nanvix/process-container/seatbelt/windows-sandbox/wslc）
- 该仓库 robots.txt 不拦截 git clone（10-23 strands-box 曾遇 robots 拦截博客替代，MXC 无此问题）
