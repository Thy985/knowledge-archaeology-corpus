# MCP（Model Context Protocol）知识考古

**一句话定位**：AI 应用与外部数据源/工具之间的**垂直集成协议**（agent↔tool 的"USB-C"）。2026-07-28 版本完成**无状态化转型**（SEP-2575）：移除 initialize 握手与会话，每请求 `_meta` 自描述，配 `server/discover` 前置协商 + MRTR 取代服务器发起请求——成为可水平扩展的企业级 Agent 基础设施。对照 A2A（水平协作协议，agent↔agent），MCP 是垂直工具协议（agent↔tool），两层互补构成 Agent 基础设施双栈。

## 考古信息
- **run_id**: ARCH-2026-09-25-001（每日 Knowledge Archaeology Runtime SOP）
- **commit**: `ab3a39c13bd23be691c2760e1c6c5c15a64582e1`（main，2026-09-24 pushed）
- **协议版本**: 2026-07-28（LATEST_PROTOCOL_VERSION）
- **skill_version**: knowledge-archaeology v3.2
- **日期**: 2026-09-25

## 产物清单
| 目录/文件 | 内容 |
|---|---|
| 00_overview.md | 考古概览 + 5 个核心认知资产 |
| 01_project-layer/ | 项目地图（架构/核心抽象/生命周期/兼容矩阵/治理/扩展体系） |
| 02_engineering-knowledge/ | 30 条 EK（EK Graph，六类边，46 出边，0 游离） |
| 03_knowledge-layer/ | 7 个 KO（R1-R4 聚合规则矩阵 + 五层阶梯） |
| 04_flow-atlas/ | 七类流（Control/State/Data/Evidence/Authority/Memory/Policy） |
| 05_candidates/ | 8 条未验证假说（含 2 条跨协议假说） |
| 06_validation/ | 盲重建验证（20 判定：15 CONFIRMED / 2 PARTIAL / 1 DOWNGRADED / 1 OVER_GENERALIZED / 1 MISSING / 0 CONTRADICTED） |
| archaeology-runs/ARCH-2026-09-25-001/ | 本 run 元数据 + job manifest + 仓库快照 |

## 关键发现（Top 5）
1. **无状态化是规模化地基**：移除 session/握手，每请求 `_meta` 自描述（版本/能力/身份）——负载均衡无 sticky session、实例故障无状态可恢复、实现无 session GC 负担（SEP-2575 "pay as you go"三原则）
2. **MRTR 重新折叠双向数据流**：服务器需要采样/用户输入/根目录时，不再发起反向请求，而是返回 `resultType:"input_required"` + inputRequests，客户端重发原请求附 inputResponses（SEP-2322）
3. **扩展治理三明治**：核心冻结 + 扩展外置化（SEP-2133）——Tasks/Skills/Apps/Auth 全部为可选扩展，默认关闭需显式 opt-in，SDK 自治实现
4. **Feature lifecycle + 废弃纪律**：三态 + 12 个月窗口（SEP-2596）；Roots/Sampling/Logging/DCR 于 2026-07-28 正式废弃；错误码空间分区（legacy 冻结/规范保留段 -32020~-32099）
5. **治理与规范同库**：SEP=PR 编号文件（44 个决策记录）、四级维护者制（防公司捕获）、AGENTS.md AI 贡献政策（<3 合并 PR 门槛 + disclosure.txt）

## 验证结论
PASS_WITH_NOTES：无 CONTRADICTED；1 项 DOWNGRADED（Elicitation 为唯一 Active 客户端能力）、1 项 OVER_GENERALIZED（扩展外置化断言强度下调，进 Candidates C-01）、1 项 MISSING（6 组未精读文件列入 coverage gaps）。跨协议假说 KO-07（垂直/水平分层）与 C-07（扩展外置化）获 MCP 侧证据支持。
