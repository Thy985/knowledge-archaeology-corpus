# Independent Validation Report — Webwright（ARCH-2026-10-05-001）

> 阶段⑤：独立 Auditor 盲重建。不把考古包当事实来源——Auditor 独立重读仓库关键文件建立 Independent Findings 后与包内结论对比。
> 禁止修改原考古产物（package/ 00-06 保持原样；修正项经阶段⑥ Reconciliation 落 corpus）。

## 0. 审计方法
- 独立抽查 6 处 Flow Edge / 判定声明（agents/default.py、models/base.py、skill_factory/update.py、environments/local_browser.py、skill_factory/route.py、skill_factory/learn.py），全部 grep 原文核对，未依赖考古包读取结果。
- 重点攻击面：单案例→Pattern、Pattern→L4、Flow Edge 真实性、fallback/override/alternate 路径、Epistemic 状态混淆。

## 1. 判定统计
| 判定 | 数量 | 项 |
|---|---|---|
| CONFIRMED | 4 | done 降级机制（B）、replay/refine-skip/credentials 语义（C）、CDP 探活自启（D）、route 可注入设计（E） |
| PARTIALLY_CONFIRMED | 3 | require_self_reflection_success 默认值（A）、_recover_answer 主场景（F）、KO-02 "workspace 是唯一记忆"措辞 |
| DOWNGRADED | 0 | — |
| OVER_GENERALIZED | 0 | — |
| MISSING | 2 | tests/unit 2 文件细节、config/{crafted_cli,task_showcase}.yaml 全文（考古包 06 §3 已自我声明） |
| CONTRADICTED | 0 | — |
| NEEDS_HUMAN_REVIEW | 0 | — |
| **合计** | **9** | 4 确认 + 3 部分确认 + 2 遗漏声明 |

## 2. 3 个成功（独立重读确证）
1. **done 降级（对应 EK-07）**：`models/base.py:114-119` 原文 "Strict-schema responses cannot have done=true with a non-empty action; tolerate it from non-strict callers by demoting `done`"——考古包描述与代码注释逐字一致。CONFIRMED。
2. **重放与入库护栏（对应 EK-29/32/57）**：`update.py` 中 `_replay` 真实存在（"each replay drives a live site for up to 240s"）、`replays.json` 写回、`existing VERIFIED skill has no replays.json — refine skipped`、`Credentials are never stored (the library may be shared/committed); replay borrows them from the incoming batch`——四条 EK 声明全部在源码原文命中。CONFIRMED。
3. **CDP 探活自启（对应 EK-03）**：`local_browser.py` 命中 `/json/version`、`/json/list`、`/json/new?about:blank`（quote 编码）三探活端点，与 EK-03 描述一致。CONFIRMED。

## 3. 3 个错误/需修正（PARTIALLY_CONFIRMED → Reconciliation）
1. **require_self_reflection_success 默认值表述不精确（A，对应 EK-17/01 层 §6）**：
   - 仓库事实：`agents/default.py:39` `require_self_reflection_success: bool = False`（**代码默认 False**）；`config/base.yaml` `agent.require_self_reflection_success: true`（**配置覆盖为 True**）。
   - 考古包 EK-17 写"require_self_reflection_success=true 时…"，含义正确但未点明"代码默认 False、base.yaml 配置开启"。与 step_limit（代码 15 / 配置 100）是同一"配置覆盖代码默认"模式，应显式并列为同一机制。
   - 修正：EK-17 补充"代码默认 False，base.yaml 配置 true"；KO 不受影响。
2. **recover_answer 主场景事实不完整（F，对应 EK-25）**：
   - 仓库事实：`learn.py:87-90` 注释 "Plain webwright solves write NO agent_response.json; the answer is in the trajectory's exit message (extra.submission / extra.final_response — always a STRING, structured only when the task asked for a format). Only a Submitted, non-empty exit yields an answer"。
   - 考古包 EK-25 按 ①agent_response.json → ②trajectory exit message → ③skip 排序正确，但未写明"webwright 主 solve 不写 agent_response.json（那是 skill_factory 执行器产物）；trajectory exit message 是主场景"。
   - 修正：EK-25 补充分支来源事实。
3. **KO-02 "唯一记忆"措辞过强（G）**：Auditor 独立重读确认 persistent_local_browser 会话 JSON（.lb_session.json）与 workspace 文件同为记忆载体；考古包 KO-02 已有"默认可丢弃、按需持久"分层（反例已并入），但"证据与中间状态落在工作区，成为…唯一记忆"表述在 persistent 会话语境下过强。
   - 修正：KO-02 措辞收紧为"默认记忆在文件系统；显式持久会话（persistent/browserbase）是例外层"。

## 4. 遗漏（MISSING，均已在考古包 06 §3 自我声明，不构成新问题）
- tests/unit/test_doctor.py、test_tool_model_routing.py 全文细节（已按测试名掌握断言语义）。
- config/{crafted_cli,task_showcase,local_browser,persistent_browser}.yaml 全文（任务展示/长活浏览器模板语义）。
- 均不影响本包核心结论；列入 Reconciliation 的 corpus 写入边界说明。

## 5. Benchmark case（可执行验证）
- **LLM-free 测试全绿**：`tests/skill_factory/test_evolve.py` 等 14 文件为无模型断言（CI 直接 `python file` 跑）。本审计未重跑（依赖环境装包），但抽查的核心断言（grade 三态、refine-skip、_norm 折叠、memorized-answer 判据、learned_library 样例约束）均在源码+测试文本中逐字确认。
- **Blind Reconstruction 交叉验证**：审计重建的"一次 run 消息流 / 技能入库链 / completion gate"三步序列与考古包 06 §1 的重建物一致 → 间接确认考古包非自洽虚构。

## 6. 审计结论
考古包 48 EK / 8 KO / 7 Flow 的核心结构经独立抽查**未被推翻**；2 处精度修正（默认值来源、recover 主场景）+ 1 处措辞收紧（KO-02 记忆载体）移交阶段⑥ Reconciliation。无 CONTRADICTED / DOWNGRADED / OVER_GENERALIZED 项；无 NEEDS_HUMAN_REVIEW 项。

---

# 修正清单（Reconciliation 输入，阶段⑥应用）
| # | 对象 | 修正 | 类型 |
|---|---|---|---|
| R1 | EK-17 | 补充"代码默认 require_self_reflection_success=False，base.yaml 配置 true；与 step_limit 同为配置覆盖代码默认模式" | 精度修正 |
| R2 | EK-25 | 补充"webwright 主 solve 不写 agent_response.json（skill_factory 执行器才写）；trajectory exit message 是主场景答案来源，始终 STRING" | 事实补全 |
| R3 | KO-02 | 措辞收紧："默认记忆在文件系统；persistent/browserbase 显式会话是例外层" | scope 收紧 |
| R4 | 06 §2 Truth | step_limit 行标注补充"self_reflection 默认同构（代码 False/配置 True）" | 一致性 |
