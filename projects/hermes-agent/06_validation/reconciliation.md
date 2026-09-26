# Reconciliation — ARCH-2026-09-19-001（hermes-agent）

> 合并 Independent Auditor 修正到 Corpus Artifact（不修改生产 Skill、不改目标仓库、不改 KnowlegeMap）。

## R-1（ID-1）新增 EK-40 记忆旁路面
- 修正：EK 层新增 EK-40（skip_memory/ignore_rules 记忆 bypass）；KO-01 反例追加"skip_memory=True 时冻结加载被跳过"。
- 证据：agent/agent_init.py:1243-1284；hermes_cli/cli_agent_setup_mixin.py:673

## R-2（ID-6）KO-06 表述修正
- 原："改进动作=letta memory-subagent；维护动作=hermes Curator"
- 改："改进（agent 意图驱动：skill_manage 自主创建/编辑技能 + memory 工具写盘）与维护（调度器策略驱动：Curator 空闲生命周期）分离"——两项目在"改进"上同构（agent 主动），在"维护"上 hermes 引入调度器自动化。

## R-3（ID-4）EK-13 补 hard_block 语义
- 修正：EK-13 追加"--force 仅覆盖 caution；dangerous+community/trusted 为 hard_block 不可覆盖；agent-created dangerous → ask 重试"。
- 证据：tools/skills_guard.py:660-673

## R-4（ID-5）F-30 精度
- 修正：tests/ 4,645 文件（其中 4,608 为 .py）——补精度，不改主体。

## R-5（ID-2）新增 EK-41 执行指导工具集中性
- 新增：EK-41（#39797：hard "use web_search" 曾覆盖 SOUL.md 并在 Blank Slate 悬空 → 执行指导文本保持 toolset-neutral）。
- 证据：agent/prompt_builder.py:500

## R-6（ID-3）新增 Candidate C-09
- 新增：C-09 HERMES_STATE_DB_GUARD_BYPASS 环境变量逃逸面——NEEDS_HUMAN_REVIEW；验证路径=确认是否设计内调试逃生舱。

## R-7（ID-7/ID-8）确认项
- KO-02 / EK-02 独立核验通过，无修改。

## Reconciliation 后质量
- EK：39 → 41（新增 EK-40/41，均带 links：EK-40 mechanism→EK-25/EK-05；EK-41 constraint→EK-28）
- KO：9（KO-01/KO-06 反例与表述更新）
- Candidates：8 → 9（新增 C-09）
- 判定统计：CONFIRMED 28 / PARTIALLY_CONFIRMED 3→2（R-4 后 F-30 视为确认）/ OVER_GENERALIZED 1（已修正）/ MISSING 2（已补）/ NEEDS_HUMAN_REVIEW 1（转 C-09）
