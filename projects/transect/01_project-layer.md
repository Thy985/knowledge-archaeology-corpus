# 01 Project Layer — Transect（ARCH-2026-10-13-001）

## 1.1 项目是什么

Transect 是一个 Python 包（pip name `transect` v0.2.4），基于 [Inspect Scout](https://meridianlabs-ai.github.io/inspect_scout/) 构建，把长时程 agent 评测的 transcript 转化为**可导航回合时间线 + 带 provenance/可靠性信息的标签 + 可导出数据表**。输入 Inspect `.eval` log 或 OpenClaw `.jsonl`，输出 HTML 报告（`scans/report.html`）+ Scout viewer（深链到源头回合）。

## 1.2 架构分层

```
用户入口: transect() / load() / render()          (api.py)
   │
   ▼
选择层: select_transcripts(one-sample/epoch)     (selection.py)
   │
   ▼
扫描层: Scout scanner 批
   ├─ 结构扫描器 ($0, 零 LLM): token_timeline / context_flush / human_intervention / eval_setup
   └─ judged 扫描器 (需 judge_models):
       ├─ decision_phases        (phases.py + phases_cohort/phases_common/phases_narrate/phases_verify)
       └─ subagent_classification (subagents.py)
   │
   ▼
帧层: frames/ (20 个 pandas 帧) + layer_frames   (frames/*.py)
   │
   ▼
可靠性层: reliability.py (Regime/Rate/CohortAgreement/Mean/LabelStats/RelabelRate/MemberCoverage)
   │
   ▼
报告层: report/ (render.py SECTION_KEYS 7 节 + reliability audit flags + Scout 深链)
   │
   ▼
scan store 持久化: scans/scan_id=*/ (parquet) + _scan.json → load() 可无扫描重读
```

## 1.3 核心模块职责

| 模块 | 职责 | 关键符号 |
|---|---|---|
| api.py | 公共 API 编排 | `transect()` / `load()` / `render()` / `_run()` / `_scanners()` |
| spec.py | 评估族配置 | `Spec` / `Phase` / `SubagentLabel` / `load_spec()`（extra="forbid"） |
| selection.py | transcript 选择 | `select_transcripts()`（one-sample 规则 + epoch 选择 + task_set 去重拒绝） |
| layers.py | 用户扩展单元 | `Layer`（scanner/frame/section/tags/audit + scanner_args）/ `validate_layers()` |
| scanners/base.py | 结构扫描器 | `token_timeline()` / `context_flush()` / `human_intervention()` / `eval_setup()` |
| scanners/phases.py | phase 分割扫描器 | `decision_phases()`（三 regime）+ `turn_digests()` + `system_prompt()` + `project_phase_turns()` |
| scanners/phases_cohort.py | 投票共识 | `vote_turns()`（modal → tie mean conf → tie order） |
| scanners/phases_common.py | judge 调用/拼接 | `call_judge()` / `stitch_phases()` / `resolve_phases()` / `gather_judge_calls()` |
| scanners/phases_verify.py | 二轮 verifier | `verify_phases()` / `select_for_verify()`（4 触发）/ `_Verdict` |
| scanners/phases_narrate.py | 叙述层 | `narrate_phases()` / `NarratorAudit` |
| scanners/subagents.py | 子代理分类 | `subagent_classification()`（基于 cohort_llm_scanner） |
| scanners/cohort.py | 队列机制 | `Member` / `Ballot` / `UnitVote` / `vote_ballots()` / `cohort_llm_scanner` / `resolve_verify()` |
| frames/ | 帧家族 | `TransectResults` / 20 frames / `layer_frame()` |
| reliability.py | 判定可靠性统计 | `Regime` / `Rate`(Wilson) / `CohortAgreement`(α/AC1) / `Mean`(N<8 无 CI) / `LabelStats` / `RelabelRate` / `MemberCoverage` |
| report/ | HTML 报告 | `render_report()` / `SECTION_KEYS` / `build_flags()` / `describe_regime()` |
| scan_status.py | 执行记录 | `ScanError` / `ModelTokenUsage` / `build_scan_status()` |
| tags.py | 标签族 | `select_tags()` / `RESERVED_TAG_COLUMNS` |
| skills.py | Claude Code skills 分发 | `install()` / `skill_source()` |
| ingestion/openclaw_telemetry_hal/ | vendored 上游 | OpenClaw `.jsonl` → transcript（detection/parse/extraction/populate_db） |

## 1.4 生命周期（一次 transect() 运行）

1. **前置验证**：`validate_section_order()`（typo section key 在 judge 花费前失败）；`validate_layers()`（畸形 layer 在 factory 运行前拒绝）
2. **spec 加载**：JSON/YAML → `Spec.model_validate`（extra="forbid"，拼错字段即抛）
3. **输入分流**：`_openclaw_files()` 识别 `.jsonl`（→ 导入 snapshot DB）vs `.eval`；混合即抛
4. **transcript 选择**：`select_transcripts()`（one-sample；epoch auto=最早成功；同 (sample,epoch) 多 task_set 拒绝）
5. **扫描批**：4 结构扫描器 + 条件 judged 扫描器 → `scout_scan()`（fresh scan，无 scan 级复用；judge 调用由 inspect-ai generate cache 去重）
6. **load() 建帧**：从 scan store 重读结果表 → 20 帧 + layer_frames + scan_status
7. **render() 出报告**：`render_report()` → `report.html`（多 epoch 时每 epoch 一文件 `report_epochN.html`）+ `_spawn_viewer()`（scout view 子进程，free port；TTY hold / 非 TTY detach）
8. **atexit 清理**：interactive 终止 viewer；非 TTY 留服务

## 1.5 三种判定 regime（互斥）

| regime | 条件 | 判定机制 | verifier 默认 |
|---|---|---|---|
| solo | 1 模型 1 roll | 单判 | on（同模型二轮，第二 prompt 流程） |
| k_roll | 1 模型 >1 roll | roll 投票驱动分区 | on（叠加上层） |
| cohort | >1 不同模型 | 逐单元多数投票 | off（多数票即校正机制） |

`cohort_members()` 生成成员；`resolve_verify(len(factory_names), verify, verify_sample)` 决定 verifier 武装。

## 1.6 帧家族（frames/）

`TransectResults`（dataclass）字段帧：token_timeline / flushes / interventions / lane_activity / transcript_info / phases / phase_turns / turn_groups / phase_turn_votes / subagents / subagent_votes / label_definitions / turn_tags + layer_frames。`builtin_frame_names()` 从 dataclass 字段导出（Layer 名不得撞内建帧名）。

## 1.7 报告结构

`SECTION_KEYS`（默认阅读顺序）：eval_setup → phase_timeline → interventions → token_telemetry → subagents → token_spend → phase_cards；**reliability audit 恒最后渲染**；run-wide "Scan execution & coverage" 块在所有 transcripts 之后渲染一次。`section_order` 可重排（命名节优先，未命名跟随默认序）。

## 1.8 可靠性审计（report/reliability.py）

对每个 judged 实体（phases / subagents / 自定义 audit 层）产出 `Flag`（metric/value/level amber|red/explanation/remediation），阈值常量模块级集中：

- k_roll 自洽均值 `< AGREEMENT_RELIABLE(0.8)` → amber；`> KROLL_SUSPICIOUSLY_HIGH` → amber（警惕 cache 重放/共享 prompt/名称锚定）
- cohort α `< AGREEMENT_TENTATIVE(0.66)` → red；`[0.66, 0.80)` → amber
- **任何 random-sample spot-check overturn → 必 red**
- **任何 unjudged 单元 → 至少 amber**
- same-model verifier caveat：同模型二轮不是"未变程序下的可重复性"度量

## 1.9 配置体系

pyproject：ruff（select E/F/I/B/UP/RUF，排除 vendored hal、RUF001 允许排版字符）；mypy/pyright（files=src/transect，exclude hal）；pytest（testpaths=tests）；pre-commit；uv。运行时配置：`transect()` 参数（judge_models/k_rolls/verify/verifier_model/verify_sample/max_processes/section_order/sensitivity）+ spec 文件（phases/subagent_labels/context/extra）。

## 1.10 外部依赖

inspect-ai / inspect-scout / inspect-viz（Inspect 生态核心）；pandas / pydantic / pyyaml / markdown-it-py / jinja2（报告渲染）；anthropic / openai（dev，judge 模型 SDK）；playwright（ui-test）。系统级：`scout` CLI（viewer）。

## 1.11 边界与例外

- 单次运行扫描**一个 sample**（多 sample log 必须显式 sample=，错误列出可选 id）
- decision_phases **无 open-vocabulary 模式**：spec.phases 为空即 ValueError
- OpenClaw 导入：span 结束为**合成**（ends_recorded=False）；provider 失败占位 turn（`[assistant turn failed...]`）不参与 digest；scaffold 消息（heartbeat poll）跳过
- judge 拒绝/无答案：refusal / no_answer 状态保留（basis 列），chunk 级 last_phase 重置避免假邻接
- fill 置信度固定 0.3（gap 继承标签）；missing_turn 无标签可继承时留空

## 1.12 宿主生态补充（Observation）

本仓库宿主 = UK AISI 官方；`inspect-scout` / `inspect-ai` / `inspect-viz` 均为同一生态轨道（Inspect）。`AGENTS.md` 与 `.claude/`（Claude Code skills：using-transect / transect-diagnostics / add-a-layer / custom-ui）表明官方把"AI 辅助使用/扩展工具"作为一等分发渠道（`python -m transect.skills install` 幂等复制进项目 `.claude/skills/`）。
