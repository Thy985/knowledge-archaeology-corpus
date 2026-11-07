# Reconciliation Report

> 合并独立验证（`independent_validation.md`）与生产方自评的冲突，记录每一项的处理决定。**原考古产物已按本报告修正**（这是阶段⑥允许的唯一例外：修正 Corpus Artifact，不改生产 Skill）。

## 冲突处理表

| # | 冲突 | 生产方原判 | Auditor 判定 | 处理 | 理由 |
|---|---|---|---|---|---|
| R-01 | 检索配方数量 | 13 | 16 | **采纳 Auditor**，改 EK-14 / E-31 | `COMBINED_*` 3 个常量被漏计，源码可枚举 |
| R-02 | MCP 工具数量 | 12 | 13 | **采纳 Auditor**，改 E-48 / architecture | `get_status` 带 `@mcp.tool()` |
| R-03 | 仓库规模 | 424 / 302 / 77,773 / 83 | 386 / 300 / 76,764 / 81 | **采纳 Auditor**，改用可复现口径并注明 | 初版口径不可复现 |
| R-04 | KO-04 层级 | Principle | OVER_GENERALIZED | **降为 Pattern** | 3 个回归同项目同团队同栈，非跨项目 |
| R-05 | KO-06 层级 | Principle | OVER_GENERALIZED | **降为 Pattern** | 仅 2 份文档，无跨项目对照 |
| R-06 | KO-07 层级 | Principle | OVER_GENERALIZED | **降为 Pattern + 措辞收紧** | 单一文档 + 未实测投毒 |
| R-07 | EK-03 "双时间" | 直接使用术语 | PARTIALLY | **加 audit_note**：分析术语，非仓库用词 | 避免术语冒充项目定义 |
| R-08 | EK-10 "重叠即共存" | 表述为重叠即共存 | PARTIALLY | **改为"无法证明先后则不裁决"** | 实现判据更宽：时间缺失也不裁决 |
| R-09 | EK-12 Saga 增量 | 陈述为事实 | PARTIALLY | **降为 Hypothesis，移入 CAND-06** | 未运行验证乱序场景 |
| R-10 | EK-19 重构完成度 | "部分落地" | PARTIALLY | **只陈述可观察事实** | spec 为 Draft，完成度不可量化 |
| R-11 | EK-33 "同一根因" | 断言同一根因 | PARTIALLY | **改为"共享同一类前提"** | 根因归属需 upstream 确认 |
| R-12 | 反例预算 | 8/24（自报） | MISSING | **保留缺口，不补造** | 不弱化证据标准换全绿 |
| R-13 | Flow Atlas 54 条 | 全部断言 | 抽样 12 条确认，其余未逐条 | **保留但标"抽样复核"** | 不声称全量已验证 |
| R-14 | Coverage 外围 | 已自报未覆盖 | 确认遗漏 | **保留缺口声明** | 不扩大结论范围 |

## 未采纳项

| 项 | Auditor 建议 | 未采纳理由 |
|---|---|---|
| 把 KO-01/02/03/05 也降为 Observation | 无此建议 | Auditor 判 CONFIRMED，维持 Pattern |
| 删除反例不足的 KO | 无此建议 | 降级/标注即可，删除会丢失 EK 底座 |

## 修正后的最终状态

| 维度 | 修正后 |
|---|---|
| Facts | 53（E-03/E-31/E-48 内容修正，编号不变） |
| EK | 33（EK-14 数字修正；EK-03/10/12/19/33 表述收紧） |
| KO | 8（**最高层级 L3 Pattern**；0 Principle） |
| Candidates | 7（CAND-06 由 EK-12 降级并入） |
| Flow Edges | 54（抽样 12 条独立确认） |
| EK Graph | 33 节点 / 95 边 / 平均出度 2.88 / 孤立 0% / 单连通分量 |
| 运行验证 | 0 |

## 对生产 Skill 的启示（不构成改进提案）

本轮暴露的是**编排层**问题，而非 skill 契约缺陷：单 Agent 顺序执行时，取证与计数容易在同一上下文自我确认（13 vs 16、12 vs 13、规模数字）。若后续要提 Skill Evolution，候选方向是"**计数类事实必须由脚本枚举，禁止人工清点**"。但单轮样本不足以判定为系统性缺陷——**本轮不提 Proposal**，留作观察项。
