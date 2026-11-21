# Reconciliation — ARCH-2026-10-22-001（agent-aegis）

> 合并 Independent Validation 修正。只修正 Corpus Artifact，不修改生产 Skill。

## 修正清单

### R-1 · 新增 EK-G19 · AEGIS_* 环境变量配置面（采纳 S-1 + E-3）
- **新增 EK**: env 配置族 = 第二配置入口：`AEGIS_INSTRUMENT`（1/true/yes/on → import 时自动插桩）+ `AEGIS_ON_BLOCK`（默认 **"raise"**——fail-closed 语义）+ `AEGIS_AUDIT` + `AEGIS_GUARDRAILS`（默认 "default"，暗示内置默认 guardrail 集）；与 YAML 策略配置并存
- **links**: mechanism↔EK-G02（接入梯度族）；causal→EK-G10（fail-safe 默认）
- **证据**: instrument/__init__.py:349-365（_maybe_auto_instrument 完整实现）

### R-2 · EK-G02 补 patch 架构细节（采纳 S-2）
- **修正内容**: 追加"patch 定义于 instrument/_*.py 子模块（10 框架）、`__init__.py` 逐对 re-export；每框架成对 unpatch_*；FrameworkPatch 状态对象 + InstrumentationState；README '12 frameworks' = 10 框架 + 2 raw API（integrations/patch_anthropic.py + patch_openai.py）"
- **证据**: instrument/__init__.py:43-57,62-90；integrations/patch_anthropic.py:174 + patch_openai.py:175

### R-3 · EK-G06 补第三生成器（采纳 S-3）
- **修正内容**: autopolicy_llm.py 共 3 处 generate（OpenAI:181 / Anthropic:257 / 第三生成器:339，类型未深读标注）
- **证据**: autopolicy_llm.py:181,257,339

### R-4 · EK-G16 标注默认 guardrail 集未深读（采纳 E-2）
- **修正内容**: "空配置跳过（cfg is None → return）/ 初始化失败 warning 不阻断"行为 CONFIRMED；**默认 guardrail 集构成未深读**（AEGIS_GUARDRAILS="default" 暗示存在）
- **证据**: init.py:225-235（_setup_guardrails）

### R-5 · 06_validation 缺口收窄（采纳 E-1/E-3）
- **修正内容**: "12 frameworks 计数口径"改为"Auditor 已核实成立（子模块 10 + raw API 2）"；测试徽章（6400+/92%）保留为"未独立复跑"缺口声明

## 未修改项
- KO-G01~G05、Flow Atlas、Candidates、01/03/04/05 主体（无攻击成立）
- 原考古产物不可修改原则：以上修正以 corpus 落盘版本为准，本地 package 保留初版 + 本文件记录差异

## Reconciliation 后 EK 统计
- EK 总数: 19（G01~G19）；links 覆盖率 100%；平均出边 ~1.8；游离 0%
- 6 项修正全部可反向追溯独立证据（independent_validation_report.md）
