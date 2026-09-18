# Independent Validation Report — ARCH-2026-09-19-001（hermes-agent）

> Auditor 独立重读仓库（未以考古结果为准），建立 Independent Findings 后与原考古对比。
> 禁止修改原考古产物；本报告驱动 Reconciliation。

## 判定统计
| 判定 | 数量 | 对象 |
|---|---|---|
| CONFIRMED | 28 | F-03/04/05/07/08/11/12/13/14/17/18/19/20/21/22/23/24/25/26/27/31/32/34；EK-01/02/05/06/07/09/13/17/22/24 |
| PARTIALLY_CONFIRMED | 3 | F-30（测试数字精确化）；KO-04（--force 语义需收紧）；KO-05（状态韧性族未含 skip_memory 面） |
| DOWNGRADED | 0 | — |
| OVER_GENERALIZED | 1 | KO-06（"改进动作"表述归因不准） |
| MISSING | 2 | 独立发现 ID-1（skip_memory/ignore_rules bypass）、ID-2（#39797 工具指名教训） |
| CONTRADICTED | 0 | — |
| NEEDS_HUMAN_REVIEW | 1 | ID-3（HERMES_STATE_DB_GUARD_BYPASS 环境变量逃逸面） |

## 独立发现（Independent Findings）

**ID-1（MISSING）skip_memory / ignore_rules bypass**
- 证据：agent/agent_init.py:1243-1284（`skip_memory` 跳过外部 provider；enabled memory toolset 仍加载 MEMORY.md）；hermes_cli/cli_agent_setup_mixin.py:673（`skip_memory=self.ignore_rules`）；agent_init.py:1256（cron agents 现 run with skip_memory=False）
- 含义：存在"跳过记忆 provider"的显式 bypass 路径（CLI ignore_rules 标志）——考古 F-25/F-26 未覆盖此 bypass 面；记忆冻结快照仍有条件性（非绝对）。
- 处置：新增 EK/反例，并入 Reconciliation。

**ID-2（MISSING）#39797 工具指名教训**
- 证据：agent/prompt_builder.py:500（`#39797: a hard "use web_search" overrode SOUL.md and dangled in Blank Slate` → execution_guidance 工具集中性）
- 含义：执行指导文本指名具体工具曾覆盖 SOUL.md 并在无该工具集时悬空——这是"人格/指导层级冲突"的工程教训，考古未采。
- 处置：新增 EK-40，并入 Reconciliation。

**ID-3（NEEDS_HUMAN_REVIEW）HERMES_STATE_DB_GUARD_BYPASS**
- 证据：hermes_state_guard.py:23（`_STATE_DB_GUARD_BYPASS_ENV = "HERMES_STATE_DB_GUARD_BYPASS"`）
- 含义：生产状态库 guard 有环境变量级逃逸面。考古 F-15 提到符号但未评估风险面。是否属设计内调试逃生舱需 Owner 确认。
- 处置：写入 Candidates/反例，标注 NEEDS_HUMAN_REVIEW。

**ID-4（PARTIALLY_CONFIRMED）--force 语义收紧**
- 证据：tools/skills_guard.py:660-673（`force overrides every block except a dangerous verdict`；hard_block=dangerous+community/trusted；agent-created dangerous=ask 可重试）；:792（critical→dangerous）
- 含义：考古 F-08/KO-04 说"caution 可 --force 覆盖"正确，但未强调 **dangerous hard_block 不可 --force 覆盖** 与 agent-created 的 ask 语义。收紧表述。
- 处置：EK-13 补 hard_block 语义。

**ID-5（PARTIALLY_CONFIRMED）测试规模**
- 证据：git ls-files tests/ = 4,645 总文件，其中 4,608 为 .py
- 含义：F-30 数字正确，补精度即可。

**ID-6（OVER_GENERALIZED）KO-06 改进动作归因**
- 证据：tools/skill_manager_tool.py:2（"agent-managed skill creation & editing"）
- 含义：hermes 的"改进动作"不只在 letta（memory-subagent 反射）；hermes 自身 agent 通过 skill_manage **自主创建/编辑技能**（程序性记忆）。KO-06 表述"改进动作=letta，维护动作=hermes Curator"归因不准。
- 处置：修正 KO-06 为"改进（agent 意图驱动：skill_manage/memory 工具）与维护（调度器策略驱动：Curator）分离"。

**ID-7（CONFIRMED 强化）唯一承重边界**
- 独立核验：SECURITY.md §2.2 原文 + §3 报告范围界定 + file_safety "NOT a security boundary"——三处一致，无代码把 in-process 防御当边界。KO-02 成立。
- 附注：checkpoint shadow git / approval 门 / skills_guard 均为"协作性拒绝+审计"类，无一处自称边界——与 KO-02 无矛盾。

**ID-8（CONFIRMED 强化）prompt cache 神圣性驱动设计**
- 独立核验：AGENTS.md 不变量 1 原文 + memory_tool 冻结 + skill_manager deferred 语义 + curator auxiliary fork 不碰主 cache——四处独立实现同源同一约束。KO-01/EK-02 成立。

## 3 个最重要的成功（原考古）
1. **KO-02 唯一承重边界原则**——准确识别 SECURITY.md §2.2 并上升到"in-process 防御非边界"的认知模型；与 letta fail-closed 沙箱形成高价值跨项目对照，且独立审计未发现反证。
2. **EK-02 prompt cache 神圣性是设计因**——把 AGENTS.md 不变量识别为塑造记忆/技能/工具三系统的第一约束，这是真正的因果解释而非罗列。
3. **EK-13/14 供应链信任分级 + 诚实声明缺口**——正确捕捉 skills-guard-v5 的信任模型与"静态正则无法绑定动态目标"的自知边界。

## 3 个最重要的错误（原考古）
1. **KO-06 归因不准（OVER_GENERALIZED）**——把"改进动作"全部归给 letta 对照，遗漏 hermes 自身 agent-managed skill 创建（ID-6）。
2. **F-25/F-26 未覆盖 skip_memory bypass（MISSING）**——记忆 provider 生命周期描述完整但漏了显式旁路（ID-1）。
3. **F-15 符号提及未评估风险（NEEDS_HUMAN_REVIEW）**——HERMES_STATE_DB_GUARD_BYPASS 环境变量逃逸面未展开（ID-3）。

## 关键遗漏
- skip_memory/ignore_rules 记忆旁路（ID-1）
- #39797 工具指名→SOUL 覆盖教训（ID-2）
- --force 不覆盖 dangerous hard_block 的精确语义（ID-4）
- agent-managed skill 创建作为自我改进闭环的构成（ID-6）

## 错误升维检查
- 无单案例→Pattern 跳升：KO-01/02/06/07 均有双项目或明文架构证据；KO-08/09 主动留 L2。
- 无 Pattern→L4 冒进：仅 KO-02/06/07 升 L4，均有解释范围扩大论证。
- 无 ADR→实现事实混淆：全部 Facts 回溯源码/文档原文。

## Flow 错误检查
- Flow Atlas 7 类流全部符号级回溯通过；Memory Flow 缺 skip_memory 分支（已由 ID-1 补入 Reconciliation）。
- 无虚构 Edge。

## 新 Benchmark / Regression Case 建议
- **B-1 记忆 bypass 面**：`skip_memory=True` 时 provider 钩子零调用 + MEMORY.md 冻结加载仍生效的断言（回归 hermes 自身）。
- **B-2 注入扫描分级**：repo AGENTS.md 注入文本→BLOCK；HERMES_HOME 用户 SOUL.md 同文本→WARN+加载；distribution SOUL.md→BLOCK 的三态测试（可作 skill 跨项目 benchmark 输入）。
- **B-3 供应链 hard_block**：dangerous+community 无 --force 覆盖断言；agent-created dangerous → ask 重试断言。
- **B-4 WAL 回退**：伪造 locking protocol 标记→DELETE 回退断言（hermes_state_wal 已有 patchable 设计，可直接固化为 fixture）。
