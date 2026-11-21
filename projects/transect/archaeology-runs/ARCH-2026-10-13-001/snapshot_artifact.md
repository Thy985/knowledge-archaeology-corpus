# Repository Snapshot Artifact — Transect（ARCH-2026-10-13-001）

## 快照事实表

| 项 | 值 |
|---|---|
| run_id | `ARCH-2026-10-13-001` |
| project | Transect（长时程 Agent 评测可观测性/可审计性包） |
| repository | https://github.com/AI-Safety-Institute/transect.git |
| branch | main |
| HEAD commit | `75a7948257657f1c48b2fc1d2105fc2f6ab4074c` |
| commit message | "fix: report sections share one chart runtime and embed only their own tables (#63)" |
| commit timestamp | 2026-10-07 17:16:08 +0100 |
| 浅克隆大小 | 5.5 MB（含 .git） |
| 文件数 | 155（不含 .git；99 个 .py，12 个 .md） |
| 测试数 | 279 个 `def test_`（tests/ 29 个测试文件） |
| 版本 | pyproject `version = "0.2.4"`，requires-python `>=3.12` |

## 项目基础地图

| 维度 | 内容（证据来源） |
|---|---|
| **语言/打包** | Python 3.12+；hatchling wheel（`src/transect`）；uv.lock（pyproject.toml） |
| **入口** | `transect.transect(logs, spec, judge_models, ...)`（src/transect/api.py）→ 扫描+建帧+渲染报告+起 Scout viewer 全链 |
| **核心模块** | `api.py`（公共 API）/ `spec.py`（评估族配置）/ `layers.py`（用户扩展）/ `selection.py`（transcript 选择）/ `tags.py`（标签族）/ `scan_status.py`（执行记录）/ `reliability.py`（判定可靠性统计）/ `scanners/`（结构+judged 扫描器）/ `frames/`（20 个 pandas 帧）/ `report/`（HTML 报告 + 可靠性 audit 渲染）/ `ingestion/openclaw_telemetry_hal/`（vendored 上游导入）/ `skills.py`（Claude Code skills 分发） |
| **核心数据结构** | `Spec`（phases/subagent_labels/context/extra，extra="forbid"）；`TransectResults`（frames 家族 + layer_frames + report_paths/viewer_url）；scan store（Inspect Scout `scan_id=*` 目录 + parquet 表）；帧家族 20 个（token_timeline/phases/phase_turns/turn_groups/subagents/…，builtin_frame_names） |
| **状态** | scan store 持久化（`scans/scan_id=*/`，parquet 结果表 + `_scan.json`）；`load()` 可无扫描重读；判定状态（judge/verifier 层）落在帧列（basis/label_source/confidence/agreement） |
| **测试体系** | pytest 279 tests；fixtures：demo_scan / demo_scan_cohort / epochs_logs / parallel_logs / openclaw mini telemetry + `generate_*` 生成器脚本；重点：test_reliability（Gwet AC1 手算 cases、regime 分类、flag 阈值带）、test_token_accounting（cache 语义）、test_decision_phases、test_scan_status |
| **配置** | pyproject（ruff select E/F/I/B/UP/RUF + 排除 vendored hal；mypy/pyright；pytest testpaths）；`.pre-commit-config.yaml`；Makefile；`.claude/`（Claude Code skills） |
| **权限与治理机制** | 判定权威分层（judge→投票→verifier 改写 gate 0.6→narrator 不改标签）；reliability audit 显式 flag（amber/red + remediation）；sensitivity 敏感标记渲染；`Spec.extra="forbid"`（配置拼错即失败）；validate_layers 前置拒绝 |
| **外部依赖** | inspect-ai>=0.3.277、inspect-scout>=0.5.4、pandas>=3.0.5、pydantic>=2.13.5、pyyaml、inspect-viz、markdown-it-py、jinja2（pyproject dependencies）；dev: pytest/ruff/mypy/pyright/pre-commit/anthropic/openai；ui-test: playwright |

## 论文映射（arXiv 2610.08364）

- 论文核心主张（Transect 包实现）：①统一回合时间线对齐事件/令牌/子代理/行为标签 ②标签可追溯回源头回合（anchor_event_id → Scout deep link）③数据表可导出跨运行分析（frames + parquet）④评估族配置（spec）+ judge 分离 ⑤演示：AI R&D 评测 1300 万 token 行为阶段分析
- 仓库实现与论文对应：`decision_phases`/`subagent_classification`（行为标签）、`token_timeline`（令牌）、`subagents`（子代理）、`turn axis`（回合时间线）、`anchor_event_id`（溯源）、`reliability`/report audit（可靠性信息）、`examples/spec.yaml`（评估族配置）
- 论文作者：Toby D. Pilditch / Konstantinos Voudouris / Alexandra Abbas / Cozmin Ududec（UK AISI + Meridian Labs）

## 边界

- 浅克隆 depth=1：上游历史 commit/ADR 不可见；`main` 分支为当前快照（PR 活跃中）
- `src/transect/ingestion/openclaw_telemetry_hal/` 为 vendored 上游源码（ruff exclude），非 Transect 原创，考古标注其接口语义
- 本快照只记录仓库事实；论文实验数据（1300 万 token 案例）属论文证据，非仓库可复现内容
