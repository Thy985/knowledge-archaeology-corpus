# 06 Validation & Evidence — 盲重建验证报告

> run_id: ARCH-2026-09-25-001 ｜ 方法：Blind Reconstruction（独立 Auditor 重读仓库建立 Independent Findings，不与考古结论同源）｜ 结论：PASS_WITH_NOTES

## 1. 验证方法
Auditor 未以 00-05 层产物为输入，独立重读以下源材料后建立 Independent Findings，再与考古包比对：
- `schema/2026-07-28/schema.ts`（3197 行，重点 6-793 消息/元数据/能力段）
- `docs/specification/2026-07-28/{index,changelog,basic/index,versioning}.mdx`
- `docs/specification/2026-07-28/basic/patterns/{index,mrtr,subscriptions}.mdx`
- `docs/specification/2026-07-28/basic/transports/{index,stdio,streamable-http}.mdx`（1-165 行）
- `docs/specification/2026-07-28/server/{index,discover}.mdx`、`docs/specification/2026-07-28/deprecated.mdx`
- `docs/extensions/overview.mdx`、`seps/{1850,2133,2322,2575,2596,2640,2663}.md`（前 130 行）
- `GOVERNANCE.md`、`MAINTAINERS.md`、`AGENTS.md`、`AI_POLICY.md`、`SECURITY.md`（30-100 行）

## 2. 判定统计（20 项）

| 判定 | 数量 | 条目 |
|---|---|---|
| CONFIRMED | 15 | V-01 V-02 V-04 V-05 V-06 V-07 V-09 V-10 V-11 V-13 V-15 V-16 V-17 V-18 V-20 |
| PARTIALLY_CONFIRMED | 2 | V-03 V-08 |
| DOWNGRADED | 1 | V-12 |
| OVER_GENERALIZED | 1 | V-14 |
| MISSING | 1 | V-19 |
| CONTRADICTED | 0 | — |

## 3. 判定明细

### CONFIRMED
- **V-01** 无状态化主张：`initialize` 移除、`_meta` 必带 protocolVersion/clientCapabilities → schema.ts `RequestMetaObject`（required 数组）+ changelog #1-#3 双源确认 ✅
- **V-02** server/discover 为服务器 MUST 实现 → discover.mdx "Servers MUST implement" 原文 ✅
- **V-04** MRTR：resultType:"input_required" + inputRequests + 重发原请求 → mrtr.mdx Sequence + schema.ts InputRequiredResult ✅
- **V-05** resultType 多态 + 老服务器缺省 complete → basic/index.mdx ResultType 段原文 ✅
- **V-06** subscriptions/listen 取代 GET+subscribe → changelog #4 + subscriptions.mdx ✅
- **V-07** SSE 不可恢复（Last-Event-ID 移除）→ streamable-http.mdx "Resumable SSE streams via Last-Event-ID are not supported." ✅
- **V-09** 错误码分区（-32020/-32021/-32022）→ basic/index.mdx Error Codes 表 + schema.ts 错误常量 ✅
- **V-10** extensions 协商：map 携带 settings、默认关闭 → extensions/overview.mdx Negotiation + "disabled by default" Note ✅
- **V-11** 扩展独立演进 + 破坏性变更用新 identifier → extensions/overview.mdx Evolution ✅
- **V-13** Tasks 扩展化的三缺陷动机（握手脆弱/result 阻塞/tasks/list 授权作用域）→ SEP-2663 Motivation 原文 ✅
- **V-15** Roots/Sampling/Logging 废弃最早 2027-07-28 移除 → deprecated.mdx ✅
- **V-16** DCR→CIMD 授权演进 → deprecated.mdx + changelog Minor #7-9 ✅
- **V-17** stdio 命令执行=特性非漏洞 → SECURITY.md "Behaviors That Are Not Vulnerabilities" ✅
- **V-18** JSON Schema $ref 禁自动解引用网络 URI → basic/index.mdx JSON Schema Usage ✅
- **V-20** AI 贡献政策（<3 合并 PR 门槛 + disclosure.txt）→ AGENTS.md 原文 ✅

### PARTIALLY_CONFIRMED
- **V-03** 6×6 兼容矩阵 → versioning.mdx 有矩阵；但"era 是服务器属性非请求属性、客户端可缓存"为文档声明，无测试/实现证据 → PARTIAL
- **V-08** HTTP 取消=关闭响应流 → streamable-http.mdx Note 原文明确；但"stdio 取消=notifications/cancelled"只在 transports/index.mdx 概述层，patterns/cancellation.mdx 未完整核对 → PARTIAL

### DOWNGRADED
- **V-12** 原主张"Elicitation 为主、Sampling/Roots 为 Deprecated"作为并列三客户端能力 → index.mdx 原文为 "Elicitation (primary) / Sampling / Roots" 且 deprecated.mdx 确认后两者废弃；主张成立但把"primary"误写为并列 → 降级为"Elicitation 是唯一 Active 客户端能力"

### OVER_GENERALIZED
- **V-14** "扩展外置化趋势"在 KO-07/C-07 中接近普适断言 → 单仓库证据不足以支撑"2026 年代 Agent 协议标准路径" → 收紧为 Candidates C-01（PARTIALLY VALIDATED），不从 KO 断言

### MISSING
- **V-19** 未覆盖部分：patterns/cancellation.mdx 全文、transports/stdio.mdx 全文、streamable-http.mdx 166-740 行（SDK/反代/部署细节）、client/{elicitation,roots,sampling}.mdx、extensions/{apps,auth}.mdx、schema.ts 320-3200 行中未读段 → 标 MISSING 并列入 coverage gap

### CONTRADICTED
- 无（0 项）

## 4. Benchmark case（跨协议对照）
| 维度 | MCP 2026-07-28 | A2A 2026-08（09-24 考古） | 判定 |
|---|---|---|---|
| 分层 | 垂直（agent↔tool） | 水平（agent↔agent） | 互补成立（KO-07 候选升级证据+1） |
| 版本协商 | 每请求 _meta（无握手） | Agent Card / 能力宣告 | 不同实现、同一目标（C-02 支持） |
| 服务器→客户端交互 | MRTR 重发原请求 | 任务内回调/卡片交互 | 方向一致（C-04 支持） |
| 扩展机制 | SEP+默认关闭+SDK 自治 | 扩展协商 + 选项协商 | 收敛模式（C-01 支持） |
| 治理 | 四级维护者+SEP 即 ADR+AI 政策 | （A2A 轮已记录） | 治理显式化趋势一致 |

## 5. Reconciliation 修正清单（阶段⑥将并入 corpus）
1. V-12：03 层客户端能力表述改为"Elicitation 唯一 Active"（不涉及 02 EK 删除）
2. V-14：C-07 断言强度下调（"PARTIALLY VALIDATED" 而非"确认趋势"）；KO-07 保持 L3（跨协议假说仍标注 pending）
3. V-19：coverage gap 在 run_metadata.yaml 中登记，供后续 refresh 轮定向补齐
4. V-08：Flow Atlas F1 取消路径补注"stdio 细节见 patterns/cancellation.mdx（未全文核对）"

## 6. 质量指标（对照 A2A 轮基准）
| 指标 | 本轮 | A2A 轮（09-24） |
|---|---|---|
| EK 条数 | 30 | 29 |
| EK 平均出边 | 1.53 | 1.5 |
| 游离 EK | 0（30/30 全部有边） | 0 |
| KO | 7（L3×2 L4×5） | 7（L3×3 L4×4） |
| Candidates | 8 | 8 |
| Flows | 7 | 7 |
| 判定 | 20（15C/2PC/1D/1OG/1M/0X） | 20（16C/1PC/1OG/2M/0X） |
| 总体 | PASS_WITH_NOTES | PASS_WITH_NOTES |
