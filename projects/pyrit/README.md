# PyRIT — Knowledge Archaeology Index

**一句话定位**：微软 AI Red Team 官方开源框架（Python Risk Identification Tool for generative AI）——把 AI 红队攻击工程化为可组合、可编排、可评分、可持久化的攻击管线。

| 字段 | 值 |
|---|---|
| Repository | https://github.com/microsoft/PyRIT |
| 考古 run | ARCH-2026-10-03-001（initial） |
| Commit | `38d7e1121bacca15299bcc22992066a6f5d73e8d`（main，2026-10-02） |
| Version | 1.2.0.dev0 |
| 考古日期 | 2026-10-03 |
| Skill | knowledge-archaeology v3.2 |

## 产物清单
- `00_overview.md` — 三层速览 + Top 5 发现 + 质量指标
- `01_project_layer.md` — 项目地图（模块/抽象/数据结构/生命周期/配置/治理/依赖）
- `02_engineering_knowledge.md` — EK Graph（47 条，含 Reconciliation 新增 EK-46/47）
- `03_knowledge_layer.md` — Generalized KO（8 个，aggregation_rule R1-R4 全覆盖）
- `04_flow_atlas.md` — 七类流（33 Edge 可回溯）
- `05_candidates.md` — 假说（H-01~H-10）+ 跨项目候选（X-01~X-05）
- `06_validation.md` — 六项审计 + 盲重建 + Reconciliation 记录
- `archaeology-runs/ARCH-2026-10-03-001/` — run_metadata.yaml + independent_validation_report.md

## 关键认知（3 条）
1. **可判定判定链**：Score 是攻击判定唯一权威，undetermined 是一等状态不弱化为失败（KO-01）。
2. **注册即校验**：Registry 单入口 + 注册时校验 + 统一构造，扩展机制抗漂移（KO-02）。
3. **失败显式化**：并发攻击的部分失败/参数构建失败/重试全部显式可见（KO-05）。

## 连接
- 攻防对照第一块攻击侧拼图（corpus 防御侧 8+ 项目已考古）。
- 与 deepseek-harness（dsh-pentest 攻防 CI 主线）、Microsoft RAMPART（同厂）互证。
