# 05 · Candidates — 未验证假说（AProver）

> 不能确认的内容留在候选；Cross-project Candidate 不写成已验证 Principle。

## C1: witness-pattern 短路库的可迁移性（Cross-project Candidate）
- 假说: realism 确定性短路模式（uninitialized-lib/jv stub-disconnect/NULL-guard/USB-serial 框架不变量）是可跨项目迁移的
- 当前证据: 本项目内有效（4 模式, 均来自真实失败事件驱动, realism_checker.py 注释）
- 缺失证据: 其他项目对同一模式库的复现/命中率；jq 特有模式在其他代码库是否出现
- 验证路径: 在另一 C 代码库（如 RAMPART 工具层）复跑 realism 检查统计命中率

## C2: 测试-实现-文档漂移的系统性风险（Hypothesis）
- 假说: README 驱动的开发流程中，测试更新滞后于实现变更（EK-16 单例）是系统性风险而非偶发
- 当前证据: 1 个实例（agentic realism 默认值）；pytest 213 passed 仅 1 failed
- 缺失证据: 更多漂移实例 / 变更历史中 docs-commit 与 tests-commit 的时间差统计
- 验证路径: 遍历 git log 统计 tests/ 与 README 修改的时间差

## C3: scale-down 缩小验证与全尺寸等价性（Hypothesis）
- 假说: 参数化缩小验证（B=T=C=NH=4）的 CLEAN 结论可外推到全尺寸配置
- 当前证据: llm.c 22/30 clean（M1-M2 里程碑）; overflow-rigor 缓解 math-int 二义但非完全消除
- 缺失证据: 全尺寸配置下重复验证的对比实验
- 验证路径: B=T=C=NH=8/16 重跑 scorecard 对比

## C4: 跨项目假说——agent 判定只降级不删除（Cross-project Candidate）
- 假说: 在 agent 决策系统中，'非确定性判定只能降级、删除必须由确定性证据支撑'是可复用模式
- 当前证据: AProver(soundness_policy) + Aigis(CaMeL 无条件 DENY + 审计决策解耦, 09-27 考古) + Tafcm(受控工具/Intelligence≠Authority) 三项目收敛
- 缺失证据: 更多领域实例（金融/医疗/内容平台）; 各项目机制细节差异（DENY vs RETIER vs 受控接口）
- 验证路径: 在下一验证/防御系统考古中检查是否存在等价机制

## C5: LLM fallback 的健全性边界（Hypothesis）
- 假说: reachability 的 LLM fallback（BMC 子查询失败后）因 soundness_policy 兜底而不影响健全性
- 当前证据: LLM fallback 只影响分类（S1 reachability 判定），不授予删除权（soundness_policy 强制）
- 缺失证据: fallback 判定被下游真实使用的频率与误判率统计
- 验证路径: 统计 cex_validator 中 _check_reachability_with_llm 的调用次数与后续 outcome 分布

---
## Epistemic 声明
C1/C4 = Cross-project Candidate（需跨项目验证）；C2/C3/C5 = Hypothesis（本项目内待证）。
