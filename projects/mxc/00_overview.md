# MXC（Microsoft eXecution Container）— Project Archaeology Overview

> run: ARCH-2026-10-24-001 · skill v3.2 · HEAD `ffecec0`（v1.0.1, 2026-10-22）· mode initial · priority P0
> 认知主线：**"agent 不能成为自身的安全权威"——边界由开发者/组织声明、由 OS/引擎独立强制**（MXC 候选卡核心命题，源码考古确认）

## 项目定位（L0 Fact）
MXC 是微软开源的**沙箱化代码执行系统**（MIT，Rust 246K 行），在 Windows/Linux/macOS 上用统一 containment model + 类型化 SDK（Rust/.NET/Node）把不可信代码（模型输出/插件/工具）放进多重隔离后端（ProcessContainer/WSLC/LXC/Bubblewrap/Seatbelt/MicroVM/Hyperlight/IsolationSession/Windows Sandbox）。稳定契约 v1.0.0（schemas/stable），开发契约 1.1.0-alpha。

## 本次考古回答的核心问题
1. "策略驱动的遏制"在源码层面如何落实？→ 版本化契约（closed JSON）+ 引擎路由授权 + 后端强制三分离
2. 默认拒绝如何逐层闭合？→ 网络三默认 deny + FS fail-closed 触发 + 后端拒绝无法强制的策略 + telemetry fail closed
3. 学习模式（Learning）如何与强制（Enforcement）共存？→ deny-and-record vs audit/allow-all 双语义严格分离
4. 大厂开源 agent 治理（AGENTS.md/copilot-instructions）如何代码化？→ 架构不变量 + 测试文化 + review skill

## 认知主线（详见 03_knowledge-layer）
- **KO-M01 · fail-closed 是链条不是口号**：closed 请求 → 默认 deny → FS fail-closed → 后端拒绝 → telemetry fail-closed，每个可失败点显式闭合
- **KO-M02 · 契约/运行时/授权三轴分离**：versioning.md 权威表——"哪个字段存在"≠"哪个后端运行"≠"主机能强制什么"
- **KO-M03 · 能力边界诚实**：IsolationSession 显式全 allow / WSLc 双态 / Seatbelt profileOverride 逃生门 / proxy 限制——不能强制就拒绝，不静默假装
- **KO-M04 · Learning→Enforcement 渐进路径**：deny-and-record 先观察，permissive 显式弱化，plm 生成最小权限候选
- **KO-M05 · 单一引擎路由 + 显式扩展成本**：resolve_runner 唯一后端选择；ProbeCoverage 强制声明让"新后端"成本可见
- **KO-M06 · 治理代码化**：AGENTS.md/copilot-instructions 把"行为覆盖测试 + 不发明证据 + 复用正常路径"固化为评审标准

## 与已有 Corpus 的连接
- **strands-box**（10-23）：OS 级 agent 遏制双对照组——Box 胜策略语言+时间性+凭证网关；MXC 胜平台级多后端+组织策略+三模式
- **dogwood**（09-08）：策略语言本体（Box 运行时载体）；MXC 自带策略契约语言（版本化 JSON）
- **deepseek-harness**（validated 锚点）：sandbox/approval/permission-presets——MXC 的 containment+policy 是该理念的平台级实现
- **openclaw**（09-17）：MXC 生态首批接入者（官方博客）；agent-aegis（10-22）：进程内 guardrail 对照
