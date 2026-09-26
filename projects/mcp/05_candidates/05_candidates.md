# 05 Candidates — 未验证假说与跨项目候选

> 不能确认的内容留在候选；严格区分 Hypothesis 与已验证知识。Cross-project 候选不得写成已验证 Principle。

## C-01 扩展外置化趋势（跨协议假说，A2A 轮 C-07 的延续验证）
- 状态：**PARTIALLY VALIDATED**（MCP 侧证据充分：SEP-2133 扩展机制 + SEP-2640 Skills + SEP-2663 Tasks 移出核心 + 扩展默认关闭）
- 假说：协议核心冻结 + 新能力走扩展（metadata/identifier 外置化）成为 2026 年代 Agent 协议的标准演进路径
- MCP 证据：[SEP:2133][SEP:2640][SEP:2663][extensions/overview.mdx]；A2A 侧证据：A2A 轮 C-07（A2A 扩展机制）
- 缺失证据：第三个协议家族的独立实现；扩展→核心晋升的实际案例（Tasks 宣称"intended to be promoted"但尚未发生）
- 验证路径：观察 MCP Tasks/Skills 是否在 12-24 个月内晋升核心；观察其他协议（如 Agent2Agent 生态）是否复制扩展外置模式

## C-02 垂直/水平分层假说（跨协议，A2A 轮 KO-07 验证）
- 状态：**CONFIRMED in-pattern**（MCP=垂直工具协议与 A2A=水平协作协议在各自规范中明确；但"分层互补"作为普适命题仍需第三例）
- 假说：Agent 基础设施 = 水平协作协议（agent↔agent）+ 垂直工具协议（agent↔tool）双栈；Skills 是横跨两层的共享格式（A2A skill 卡 vs MCP skill://）
- 证据：MCP index.mdx（Host/Client/Server 垂直模型）、A2A 规范（agent 间协作）
- 缺失证据：两层协议在同一产品栈中的真实协作案例（如某 agent 通过 MCP 调工具、通过 A2A 调对等 agent 的端到端部署）

## C-03 "12 个月废弃窗口"的纪律有效性
- 状态：Hypothesis
- 假说：SEP-2596 的 12 个月最小窗口能防止"永久半废弃"（如 HTTP+SSE 自 2025-03-26 废弃至今仍在）
- 证据：SEP-2596 于 2026-07-28 生效，尚无任何功能到达 earliest removal 决策点（最早 2027-07-28）
- 缺失证据：实际移除案例；窗口内无迁移的存量实现如何处理
- 验证路径：2027-07-28 后观察 Roots/Sampling/Logging 是否按时移除

## C-04 MRTR 与 Tasks 的双轨是否会分化出第三种范式
- 状态：Hypothesis
- 假说：同步内联（MRTR）+ 异步轮询（Tasks）覆盖全部服务器-客户端交互后，"服务器发起请求"范式将永久退役；但长时任务中途多轮输入（Tasks update）与 MRTR 重试语义的边界可能模糊
- 证据：[SEP:2322][SEP:2663]
- 缺失证据：真实 SDK 中 MRTR 与 Tasks 组合使用的模式分析（如 elicitation 在 task 中途的形态）

## C-05 规范仓库的"文档即测试"质量门是否足以维持协议质量
- 状态：Hypothesis
- 假说：对 spec-first 仓库，CI（check:docs/schema/seps + 示例校验 + MDX 生成）可替代传统测试体系维持质量
- 证据：AGENTS.md 的 check/prep 门 + 13 个治理型 CI workflow + SEP-2484（conformance tests required for Final SEPs）
- 缺失证据：conformance 测试的实际执行率（本仓库无 conformance suite 源码，仅 policy 声明）
- 验证路径：跟踪 SEP-2484 落地后 Final SEP 是否都附带 conformance 测试

## C-06 AI 贡献政策的效果
- 状态：Hypothesis
- 假说：AGENTS.md 的"<3 合并 PR 门槛 + disclosure.txt"能显著过滤低质量 AI 提交，同时不阻塞正当 AI 辅助贡献
- 证据：政策文本存在（AGENTS.md/AI_POLICY.md）；无执行统计
- 缺失证据：被拒/披露提交的数量统计、政策对贡献节奏的实际影响
- 验证路径：GitHub API 拉取 org 提交数据（跨 spec/SDK 仓库）分析

## C-07 stdio 信任模型在企业环境的边界
- 状态：Hypothesis（与 C-02 关联）
- 假说：stdio 传输"客户端启动子进程 + 同权限"模型在企业 MCP 网关场景会系统性转向 Streamable HTTP + 授权框架（OAuth2），stdio 退居本地开发
- 证据：Streamable HTTP 引入后 SSE 残留重活（subscriptions/listen 只在 HTTP 有明确路径）；authorization 框架仅绑定 HTTP 传输 [T:basic/index.mdx Auth]
- 缺失证据：企业部署数据（本仓库无遥测）

## C-08 扩展默认关闭是否造成事实分裂
- 状态：Hypothesis
- 假说：扩展"默认 disabled + SDK 自治"会导致事实上的协议分裂（同一扩展在不同 SDK 支持度差异显著），client-matrix 文档只能缓解不能消除
- 证据：extensions/client-matrix.mdx 存在（说明支持度分化已被观察到）；SEP-2663 承认 SDK 维护者自治
- 缺失证据：跨 SDK 支持矩阵的量化缺口
