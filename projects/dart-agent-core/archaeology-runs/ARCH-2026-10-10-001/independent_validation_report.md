# Independent Validation Report — dart_agent_core（ARCH-2026-10-10-001）

> 独立 Auditor 盲重建：**不把考古结果当事实来源**，独立重读仓库建立 Independent Findings 后与 Package 对比。未修改任何原考古产物；修正仅记录于此，由阶段⑥ Reconciliation 合并。

## 0. 审计方法

- 独立重读：`pass_at_k.dart` / `pass_caret_k.dart` / `trial.dart` / `replay_llm_client.dart` / `stateful_agent.dart`（_executeTools 取消复查、RunJavaScript 检查、_runJavaScriptScript 全文）/ `sub_agent.dart`（_delegateTask、_copyParentHistory）/ `controller.dart` / `mcp_manager.dart`（桥工具与提示模板）/ `judge_calibrator.dart`（平均秩）/ `bin/`
- 攻击面：单案例→Pattern、Pattern→L4、ADR→实现、Flow Edge 真实性、bypass/override/exception/alternate/admin/fallback/legacy 路径、Epistemic 状态混淆
- 对比对象：`package/02_engineering_knowledge.md` / `03_knowledge_layer.md` / `04_flow_atlas.md` / `06_validation.md`

## 1. 判定统计

| 判定 | 数量 | 说明 |
|---|---|---|
| CONFIRMED | 9 | 关键声明逐字/逐结构匹配 |
| PARTIALLY_CONFIRMED | 1 | MCP 渐进披露输出模板细节未全验证 |
| DOWNGRADED | 1 | EK-41 "log-domain" 表述与实现不符 |
| OVER_GENERALIZED | 0 | 无 KO 宣称跨项目 Law |
| MISSING | 5 | 未覆盖面（已披露 + 新发现）|
| CONTRADICTED | 0 | 无实质矛盾 |
| NEEDS_HUMAN_REVIEW | 1 | RunJavaScript `../` 越界行为 |

## 2. 3 个成功（CONFIRMED 代表例）

**S-1 · EK-05 取消复查逐字匹配**
- 独立读 `_executeTools`：注释原文 "A tool may return normally after observing cancellation. **Do not let its stopFlag or a worker result turn a cancelled task into success.**"（L1909-1910）→ `if (cancelToken?.isCancelled ?? false) throw cancelToken!.cancelError!`（L1911-1913）；catch 内 "**Exception codes alone do not define task scope: a worker can stop locally. Only the shared cancellation token terminates this run.**"（L1940-1941）。
- 结论：CONFIRMED。Package 引用准确。

**S-2 · EK-45 strictReplay 默认 + CI 语义逐字匹配**
- 独立读 `replay_llm_client.dart`：`this.strictReplay = true`（L55）+ 注释 "CI default: pass `strictReplay: true` and no `fallback` so any cache miss fails the build, forcing re-recording"（L31-32）+ RecordingNotFoundException 定义。
- 结论：CONFIRMED。

**S-3 · EK-31 clone 隔离全结构匹配**
- 独立读 `sub_agent.dart`：workerSessionId=`{parent}_{assignee}_{uuid}`（L82-83）；metadata `parent_session_id`+`sub_agent_mode`（L88-90）；`_copyParentHistory` 最近 10 条（L218-220）；`<parent_agent_state_snapshot>` 包装 + ack 消息 "Got it. Thanks for the additional context! I already know the parent agent's context, I will focus on completing my task"（L222-229）。
- 结论：CONFIRMED。

## 3. 3 个错误 / 需修正（Auditor Findings）

**F-1 · DOWNGRADED — EK-41 "log-domain 迭代"表述与实现不符**
- 独立重读 `pass_at_k.dart`：注释 L17 "// log-domain product to avoid overflow on big n."；**实现为朴素连乘** `prob *= (n - c - i) / (n - i)`（L30-32），无任何 log/exp 调用（grep 确认仅 L17 一处 "log" 是注释）。
- 影响：EK-41 中 "log-domain 迭代 `1 - ∏…` 防溢出" 的"log-domain"声称不准确——实现未真正使用对数域（对大 n 的防溢出承诺未被代码兑现；实际范围 n 较小故低风险）。
- 修正建议（Reconciliation）：EK-41 改为 "朴素连乘（注释声称 log-domain 防溢出，实现实为直接连乘）"。

**F-2 · NEEDS_HUMAN_REVIEW — RunJavaScript 路径前缀的 `../` 越界行为**
- 独立读 `_runJavaScriptScript`：`fsAbsolutePath(scriptPath)` → `resolvedAbsolute.startsWith(rootWithSep)` 前缀匹配（L756-761）。
- 未验证点：`fsAbsolutePath` 是否对 `..` 做规范化（若返回原始路径，则 `<root>/../sibling.js` 仍以 `<root>/` 为前缀而逃逸白名单）。前缀匹配 + 未确认的规范化 = 潜在 bypass 路径。
- 影响：Authority Flow 中 JS 白名单的完备性未闭合。需人工审查 `core/fs_io.dart:fsAbsolutePath` 实现（后续轮补读）。
- 处置：新增 C-09（06_validation.md 已预留），阶段⑥ 写入 Candidates 而非结论。

**F-3 · PARTIALLY_CONFIRMED — EK-34 MCP 渐进披露的模板细节**
- 独立读 `mcp_manager.dart` L112-132：确认 system prompt 引导语含 "**Discovery**: Use `mcp_list_tools`…" / "**Execution**: Use `mcp_call_tool`…" / "**Resources**…" / "**Prompts**…" 四段指引 + L123-133 确认能力计数式披露意图。
- 未验证：`buildMcpSystemPrompt` 的逐字输出模板（服务器描述截断长度/计数格式）未逐行核（函数体在 L100-140 之间，我只抽样）。判定 PARTIALLY_CONFIRMED，不影响 KO-04 结论（披露策略证据充分）。

## 4. 遗漏（MISSING）

| # | 遗漏面 | 影响 | 处置 |
|---|---|---|---|
| M-1 | `message.dart`（593 行）多模态 content parts 结构 | Flow Atlas Data 流粒度（media/thought 编码）| 记 Level-2 待补 |
| M-2 | `suite_health/saturation_status.dart` 阈值常量 | EK-50 引"阈值"未给数值 | 记 Level-2 待补 |
| M-3 | `eval/loaders/suite_loader` + `json_eval_task` + `grader_registry` | 评测任务装载/grader 注册面未覆盖 | 记 Level-2 待补 |
| M-4 | `graders/code_grader`（62）/`human_grader` | grader 具体实现面 | 记 Level-2 待补 |
| M-5 | 其余 39 个测试文件 | 测试揭示行为的全量核对 | 记 Level-2 待补 |

均不改变 EK/KO 结论（主路径证据已实读），但 04 Flow Atlas 与 06 中相应边/阈值标注为"待补"。

## 5. Benchmark case（用已完成项目验证 skill 判别力）

- **B-1 · pass^k 动机对照（thinkingbox 10-08）**：两项目实现同一经验公式；thinkingbox 动机=对抗 pass@1 幻觉/区分难例，dart 注释动机=CI 报告直观一致性。独立 Auditor 确认两注释原文存在 → C-01 假说成立（同一估计器的动机反映团队报告文化）。
- **B-2 · MCP 桥对照（mcp corpus）**：dart 6 桥工具为协议层"先列目录、按需调用"的工具化封装，与 MCP 协议的能力发现原语一致 → C-04 渐进披露统一原语假说增强。
- **B-3 · judge null 对照（agentevals 未考古、thinkingbox 已考古）**：dart 显式引用 Anthropic Step 5 → 待 thinkingbox grader 层核对后闭合 C-02。

## 6. Epistemic 状态攻击结果

- 未发现 Package 中 Hypothesis 冒充 Fact（C-01~C-08 全部保留在 Candidates）。
- 未发现跨项目 Candidate 写成 Principle（KO-03/KO-05 均标 pending）。
- F-1 是唯一"表述超实现"案例（DOWNGRADED 已处置）。

## 7. 结论

- 主考古结论（KO-01~08、Flow Atlas、EK 图）**经受盲重建**：核心声明逐字可回溯。
- 需 Reconciliation 修正：F-1（EK-41 log-domain 表述）+ F-3（MCP 模板标 PARTIAL）+ F-2（C-09 入 Candidates）+ M-1~M-5（Level-2 待补清单）。
- 无实质 Contradiction；无 OVER_GENERALIZED。
