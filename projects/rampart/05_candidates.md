# 05 · Candidates（未验证 / 待定）

> 不能确认的内容留在 Candidate。Cross-project hypotheses / Tentative patterns / Scope-uncertain conclusions / Unresolved contradictions。

## C-01 · KO-01 跨项目验证（CLEAR/AgentEval 同构对照）
- **类型**：Cross-project hypothesis
- **内容**：IBM Agentic CLEAR（arXiv 2605.22608）的三级 judge（step/trace/rubric）与 RAMPART 的三态 evaluator + undetermined_operands 可能同构——都是"多级判定 + 不确定显式报告"。若成立，KO-01 从"RAMPART validated"升为跨项目 Pattern。
- **证据现状**：RAMPART 侧 S4；CLEAR 仅来自 KnowlegeMap 候选卡（未源码验证）
- **验证路径**：读 CLEAR 源码 → 对照 judge 三态处理

## C-02 · KO-02 极性分离在 promptfoo/DeepEval 中的映射
- **类型**：Tentative pattern（跨项目）
- **内容**：promptfoo 的 assertion 也是"条件检测 + 语义由断言类型决定"——与 RAMPART 的 polarity-free evaluator 同族。待验证。
- **状态**：需对照 promptfoo assertion 语义

## C-03 · gather-vs-TaskGroup 决策的通用性
- **类型**：Tentative pattern
- **内容**：RAMPART 用 gather(return_exceptions=True) 而非 TaskGroup 避免"已创建未清理"孤儿。此决策在"并发外部副作用资源"场景的普适性待跨项目验证。
- **scope 不确定**：仅当资源创建有外部副作用（远程上传）且可恢复注册清理时成立

## C-04 · XPIA 的 attack 语义中 UNDETERMINED 处理是否过保守
- **类型**：Unresolved contradiction（scope-uncertain）
- **内容**：resolve_as_attack 中 UNDETERMINED 仅当无 DETECTED 时生效；但 _adjust_for_observability 会把 SAFE 降级 UNDETERMINED。两者叠加后，"RESPONSE_ONLY + 零工具 + 无 DETECTED"→ UNDETERMINED，而"有工具调用但无 DETECTED"→ SAFE。**这是否意味着工具调用可见本身就能"洗白"攻击**？——代码如此（_adjust_for_observability 三条件含"零工具才降级"），但语义合理性存疑，需作者确认或更多反例。
- **证据**：attacks/_xpia.py:216-246 三条件；tests 锁定该行为（test_response_only_with_tool_calls_stays_safe）

## C-05 · LLMJudge 的 FULL transcript scope 成本
- **内容**：TranscriptScope.FULL 让 judge 看全对话，token 成本随轮数线性增长；无长度上限保护（除 max_turns=25）。超大对话 judge 的成本/延迟边界未定义。
- **验证路径**：成本基准

## C-06 · HarmCategory 的 StrEnum 扩展性双刃
- **内容**：HarmCategory 允许任意字符串分类（团队自定义），built-in 值只做补全/防 typo。这牺牲了枚举强类型换取扩展性——分类漂移风险（同义不同名）未治理。
- **类型**：Scope-uncertain conclusion

## C-07 · incomplete 强制的边界（Reconciled）
- **内容（修正）**：`_enforce_incomplete_exit_status` 在 sessionfinish **无条件调用**（非 xdist 也走）——xdist worker 丢失**和**序列化超限截断都会设 `is_incomplete` → 强制非零退出。真正的缺口收窄为：`_absorb_results` 的 except 分支只 warning "results may be incomplete"、**不设 incomplete、不强制失败**——单进程下"吸收失败但测试全绿"仍可能静默通过。
- **类型**：Tentative gap observation（范围已收窄）

## C-08 · payload 二进制格式（PDF/DOCX）转换链的证据缺口
- **内容**：converters 只实现 DocxConverter；PDF/XLSX/AUDIO 的 PayloadFormat 已定义但无转换器实现。多格式 XPIA（PDF 注入）路径未完整验证。
- **证据**：rampart/converters/__init__.py 仅 DocxConverter；types.py PayloadFormat 含 PDF/XLSX/AUDIO
- **类型**：Unresolved gap（Feature incomplete）

## C-09 · Clarity（companion 工具）未在本次 run 考古
- **内容**：RAMPART 与 Clarity 同批发布（同一 MS 博客），但 Clarity 是设计审查助手（.clarity-protocol/ 目录），架构与 RAMPART 完全不同。本次 run 仅 RAMPART（每天 1 项目约束）。
- **类型**：Scope boundary（明确的未考古项）
