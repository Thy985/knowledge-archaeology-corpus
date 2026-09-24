# MCP（Model Context Protocol）知识考古 — 00 Overview

> run_id: ARCH-2026-09-25-001 ｜ commit `ab3a39c` ｜ 协议版本 2026-07-28 ｜ knowledge-archaeology v3.2
> 仓库类型：**协议规范库**（spec-first，无运行时源码/无测试体系，956 文件）

## 一句话定位
MCP 是 AI 应用与外部数据源/工具之间的**垂直集成协议**（agent↔tool 的"USB-C"），2026-07-28 版本完成**无状态化转型**，成为可水平扩展的企业级 Agent 基础设施；与之对照，A2A（2026-09-24 已考古）是 agent↔agent 的水平协作协议。

## 核心认知资产（Top 5）
1. **无状态化（stateless-first）**：SEP-2575 移除 initialize 握手与 session，请求自带 `_meta`（协议版本/客户端能力/身份）；"pay as you go"三原则——优先无状态 → 状态引用 → 状态化最后手段
2. **MRTR 取代服务器发起请求**：SEP-2322 用 `InputRequiredResult` + retry 原请求承载 sampling/elicitation/roots；服务器不再主动发请求（SEP-2260 强制关联）
3. **扩展治理三明治**：核心冻结 + extensions 外置化（SEP-2133）——Tasks/Skills/Apps/Auth 全部为可选扩展，`extensions` 能力协商 + 优雅降级；扩展默认关闭需显式 opt-in
4. **Feature lifecycle + 废弃纪律**：SEP-2596 三态（Active/Deprecated/Removed）+ 12 个月窗口；Roots/Sampling/Logging/DCR 于 2026-07-28 正式废弃
5. **治理即协议的一部分**：SEP-932 四级治理（Lead→Core→Maintainer→Contributor）+ SEP-1850 PR 工作流 + AGENTS.md AI 贡献政策——协议标准与项目治理在同一个仓库中版本化

## 与已有 Corpus 的连接
- **A2A（09-24 考古）**：MCP=垂直工具协议 vs A2A=水平协作协议——直接验证 KO-07（分层认知）与 C-07（扩展外置化趋势）跨协议假说
- **OpenClaw（skills 兼容体系）**：Skills over MCP（SEP-2640）把 skill 定义为 MCP server 的规范路径
- **skillfortify**：skill 供应链安全（skill:// URI、嵌套技能 fresh consent、allowed-tools 审批门）与 AST Top10 治理呼应

## 产物导航
| 目录 | 内容 |
|---|---|
| 01_project-layer/ | 项目地图（架构/核心抽象/生命周期/配置/治理/依赖） |
| 02_engineering-knowledge/ | EK Graph（29 条，六类边） |
| 03_knowledge-layer/ | 7 个 KO（R1-R4 聚合规则矩阵 + 五层阶梯） |
| 04_flow-atlas/ | 七类流（Control/State/Data/Evidence/Authority/Memory/Policy） |
| 05_candidates/ | 8 条未验证假说（含跨协议假说） |
| 06_validation/ | 盲重建验证（20 判定） |
