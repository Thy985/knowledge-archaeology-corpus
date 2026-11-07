# Candidates · 未验证的假设与暂定模式

> 全部标 `Hypothesis`。**不得**在 Knowledge Layer 中当作已验证知识引用。

## CAND-01 「显式负向清单」是可复用的工程治理模式
- **status**: Hypothesis · **scope**: cross-project candidate
- **主张**: 成熟工程仓库会把"已知不安全/不支持/不在范围内"的能力**显式写成负向清单**（而非留白或免责声明），从而阻止使用者误判能力边界。
- **本项目证据**: `SECURITY.md` 把 6 项缺失能力 + 3 类 out-of-scope 报告类型逐条列出；Kuzu extra 注释"上游停维护，将移除"。
- **跨项目线索（未验证）**: `go-sdk`（2026-11-07 run）在文档中标注 `rough_edges` / conformance baseline；corpus 中 `durare` 亦有能力边界声明。**尚未做逐条对照**。
- **缺失证据**: 需要 ≥2 个项目的负向清单做结构化比对（条目类型、是否可执行、是否被 CI 校验）。
- **验证路径**: 在 corpus 中抽样 3 个项目，比较"负向清单 → 代码实现 → 测试"三者是否一致。

## CAND-02 记忆投毒防御的有效位置在「写入准入」，不在「检索过滤」
- **status**: Hypothesis · **scope**: cross-project candidate（mem0 / graphiti / letta）
- **主张**: 当记忆系统允许不可信来源与可信来源共用同一分区时，任何检索侧过滤都无法撤销已写入的污染；有效控制点只能是写入前的来源分级。
- **本项目证据**: `SECURITY.md` 明示同 group 同等信任、应用层负责选择内容与 group（E-51）。
- **缺失证据**: 未实测"投毒后检索是否被污染"；未对照 Mem0 的写入策略。
- **验证路径**: 构造 2 条冲突 episode（一真一假）写入同 group，观察检索返回；再对照分 group 写入的差异。

## CAND-03 「双时间 + 溯源」可能成为长期记忆系统的分层判据
- **status**: Hypothesis · **scope**: cross-project candidate
- **主张**: 可按"是否维护 `valid_at/invalid_at` + provenance 链"把记忆系统分为"快照型"与"断言型"两类，两类适用的查询/审计场景不同。
- **本项目证据**: Graphiti = 断言型（E-10）。
- **反例线索**: Mem0 v3 为 ADD-only、不维护失效时间（见 KnowlegeMap 的 S 卡），仍被广泛使用 → 分类不是优劣判断。
- **缺失证据**: 需要 3+ 系统的字段级对照。
- **验证路径**: 建立字段对照表（时间字段数、provenance 粒度、删除语义）。

## CAND-04 「有条件全绿」是测试治理的普遍现实
- **status**: Hypothesis · **scope**: cross-project candidate
- **主张**: 大型仓库的"CI 全绿"常依赖 DISABLE 组合与 ignore 列表；直接引用"测试通过"会系统性高估覆盖。
- **本项目证据**: `make test` 需 Neo4j、11 个既有失败文件不跑、无 key 时 exit 5（E-53）。
- **缺失证据**: 其他项目的 `--ignore` 列表规模未统计。
- **验证路径**: 对 corpus 项目抽样，统计 CI 命令中的 ignore/disable 项数量。

## CAND-05 `group_id` 接近"租户"，但不是安全边界
- **status**: Hypothesis · **scope**: project-internal，跨项目待验证
- **主张**: `group_id` 提供数据分区与路由，不提供身份/授权；把它当租户边界会在多调用方场景失效。
- **本项目证据**: A-01/A-02（无鉴权、clear_graph 可直接清空）。
- **缺失证据**: Zep 托管侧如何把 group 映射到用户/项目，未获取（无账号）。

## CAND-06 Saga 增量总结的"增量"边界未完全确认
- **status**: Hypothesis · **scope**: project-internal
- **主张**: `summarize_saga` 只处理"上次总结后新增的 episode"，依据是 `last_summarized_at` / `last_summarized_episode_valid_at` 与 `_summary_episodes_skipped` 计数。
- **证据**: `nodes.py:907-918` 字段 + `graphiti.py:590` 入口。
- **未确认**: 是否在 episode 乱序（reference_time 回退）时仍正确跳过；需运行验证。
- **验证路径**: 构造乱序 episode 序列，检查 summary 覆盖范围。

## CAND-07 Kuzu 后端的移除时间表未知
- **status**: Hypothesis · **scope**: project-internal
- **主张**: `pyproject.toml` 注释"上游停维护、未来移除"，但驱动代码仍在、无 deprecation warning 路径。
- **缺失证据**: 无 issue/roadmap 明确时间点（浅克隆无 issue 历史）。
- **风险**: 使用 Kuzu 的调用方可能在无预警情况下失去支持。
