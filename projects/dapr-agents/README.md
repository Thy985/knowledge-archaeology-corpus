# Dapr Agents — Knowledge Archaeology Index

**一句话定位**：CNCF Dapr × NVIDIA 的分布式 Agent 运行时（Python）——把 Dapr 的 durable workflow 引擎 + actor + state store + pub/sub 搬到多 agent 系统，agent 每步推理与工具调用持久化，崩溃后从最后完成步骤恢复、不重放已完成 LLM/工具调用。

| 字段 | 值 |
|---|---|
| Repository | https://github.com/dapr/dapr-agents |
| 考古 run | ARCH-2026-10-30-001（initial） |
| Commit | `1fd68b36`（main，2026-10-27 "fix(deps): bump locked dependencies…"） |
| Version | v1.1.0（PyPI 2026-10-30 实测） |
| 考古日期 | 2026-10-30 |
| Skill | knowledge-archaeology v3.2 |

## 产物清单
- `00_overview.md` — 六条认知主线 + durable execution 四路线对照 + 审计声明
- `01_project_layer.md` — 项目地图（架构/crate 级模块/数据结构/生命周期/状态机/配置/治理）
- `02_engineering_knowledge.md` — EK Graph（20 条，平均出边 2.35）
- `03_knowledge_layer.md` — Generalized KO（7 个，aggregation_rule 全覆盖，L3×2/L4×4/L5×1）
- `04_flow_atlas.md` — 七类流（28 Edge 可回溯）+ Flow→KO 交叉校验 7/7
- `05_candidates.md` — 跨项目假说 3（步骤持久化 vs 状态挂起 / HITL 与重放兼容 / 工具副作用可重入）+ 暂定 3
- `06_validation.md` — 六审计结果 + 质量指标
- `archaeology-runs/ARCH-2026-10-30-001/` — run 原始产物（job_manifest/snapshot_artifact/run_metadata/IVR）

## 核心发现（Top 3）
1. **Workflow 即执行骨架**：executor 回合 = activity、HITL 审批 = wait external event（确定性 uuid5 关联）、子 agent = child workflow——agent 控制流转译为可重放步骤序列（EK-D03/D04/D05）
2. **重放安全纪律**：非确定性 I/O 必须 activity 化 + 确定性 ID + WorkflowReplayFilter 抑制重放日志；破坏确定性 = 恢复出错（EK-D03/D07）
3. **双 state store + 会话至少一次**：agent-memory（会话历史）与 agent-workflow（durable state）分离；executor 重试恢复会话、已执行工具可能重跑 → 工具副作用必须可重入（EK-D06/D08）

## 验证结论
IVR（盲重建）：12 CONFIRMED / 1 PARTIALLY_CONFIRMED / 0 CONTRADICTED / 4 MISSING；Reconciliation 合入 EK-D03 工具双路径边界 + EK-D05 uuid5 确定性；无 NEEDS_HUMAN_REVIEW
