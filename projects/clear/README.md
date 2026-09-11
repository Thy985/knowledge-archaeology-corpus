# CLEAR — Knowledge Archaeology

IBM Research 开源的 **LLM 错误分析工具包**（Apache-2.0，Python，PyPI `clear-eval` v2.0.5）：用 LLM-as-a-Judge 把 agent 轨迹转成可核查证据链——**评估是证据生产管线，不是打分器**。与个人 Validation 编译器（Claim→Evidence→Judgment）直接同构。

## 关键结论（速览）

- **评估管线 = 证据生产**：trace → 统一 IR → compact → 多粒度评估（task_success/full_trajectory/rubric）→ 聚合（issues/root_cause）→ 统计模式挖掘 → dashboard
- **中间表示解耦**：provider/framework agnostic 的 IR CSV 是唯一稳定契约，下游零感知来源框架；新增来源 = 新预处理器
- **统计严谨性内嵌**：find_predictive_patterns 用 Fisher exact + Benjamini-Hochberg 校正 + effect/lift——模式挖掘是假设检验
- **judge 输入纯度**：系统消息显式"只依据轨迹内容，禁止外部元数据影响"
- **评测工程化**：温度探测回退（模型拒绝 temp=0 时自动禁用）、断点续跑 checkpoint、缓存、并发、外部 judge 插件化
- **失败根因只在失败时采集**——评估预算投向失败样本

## 考古时间线

| Run | 日期 | 模式 | commit | 结果 |
|-----|------|------|--------|------|
| ARCH-2026-09-10-001 | 2026-09-10 | initial | 9a5367b | ✅ 合并（PR #10） |

## 质量证据

- **测试实测（S4）**：`pip install -e .` + `pytest` → **211 passed + 3 skipped in 10.27s**（8 测试文件 2758 行）
- Source Fidelity：候选卡主张完整实证（对照 OpenCode 轮失真 10 倍）
- 独立验证：24 CONFIRMED / 2 PARTIAL / 0 错误升维 / 0 NEEDS_HUMAN_REVIEW

## 产物清单

- `00_overview.md` / `01_project-layer.md` / `02_engineering-knowledge.md`（27 EK）/ `03_knowledge-layer.md`（6 KO + 4 CM + 3 M）/ `04_flow-atlas.md`（七类流）/ `05_candidates.md`（8 候选）/ `06_validation.md`
- `archaeology-runs/ARCH-2026-09-10-001/` — 原始 run 快照

## 开放项

- C-04（dashboard 统计模式的下游消费）、C-05（config merge 无独立测试）、C-08（IBM 生态依赖影响）
