# Infinity — Knowledge Archaeology Index

**一句话定位**："agent 即数据"的极低内存并发挂起 runtime——step 间 agent 无进程、无栈、无连接，挂起免费、事件驱动恢复、serverless 规模到零。Rust · Apache-2.0 · Hydro 项目（AWS × UC Berkeley）。

| 字段 | 值 |
|---|---|
| Repository | https://github.com/hydro-project/infinity |
| 考古 run | ARCH-2026-10-29-001（initial） |
| Commit | `1abc6fcff1f186124f924b56aa92c3a3684c90c5`（main，2026-10-27 "fix: enable aws-config sso feature…"） |
| Version | sub-crates v0.2.0（10-22） |
| 考古日期 | 2026-10-29 |
| Skill | knowledge-archaeology v3.2 |

## 产物清单
- `00_overview.md` — 六条认知主线 + 关键数字 + Corpus 连接（三路线对照）+ 审计声明
- `01_project_layer.md` — 项目地图（crate 分层/核心模块/数据结构/生命周期/状态机/配置/治理）
- `02_engineering_knowledge.md` — EK Graph（24 条，links 六类边，平均出边 2.75）
- `03_knowledge_layer.md` — Generalized KO（8 个，aggregation_rule R1-R4 全覆盖，L3×2/L4×5/L5×1）
- `04_flow_atlas.md` — 七类流（34 Edge 可回溯）+ Flow→KO 交叉校验 8/8
- `05_candidates.md` — 跨项目假说 3（可恢复执行单元/commit-before-side-effect/agent 即数据）+ 暂定 3 + 已排除 1
- `06_validation.md` — 六审计结果 + 质量指标 + 待 IVR 攻击面
- `archaeology-runs/ARCH-2026-10-29-001/` — run 原始产物（job_manifest/snapshot_artifact/run_metadata/IVR）

## 核心发现（Top 3）
1. **Slice 有界执行 + 双屏障**：一步=加载→补全→持久化→≤1 异步工具→结束；commit barrier 保证副作用因果先 durable（EK-D01/D02）
2. **消息 ID 幂等 + 反向走查**：at-least-once 投递上的正确性靠 processed IDs + 历史尾走查，不依赖恰好一次（EK-D05/D06）
3. **Agent 即数据（L5）**：状态纯数据化 → 同一代码跑 resident 与 serverless（规模到零）——与 restate/dbos 构成 durable execution 三路线（KO-D05，Cross-project pending）

## 验证结论
IVR（盲重建）：14 CONFIRMED / 1 PARTIALLY_CONFIRMED / 0 CONTRADICTED / 5 MISSING；Reconciliation 合入 EK-D02 sync 分支 + EK-D03 未知工具路径；无 NEEDS_HUMAN_REVIEW
