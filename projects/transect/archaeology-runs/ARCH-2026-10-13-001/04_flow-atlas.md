# 04 Flow Atlas — Transect（ARCH-2026-10-13-001）

> 七类流，全部从真实代码导出；关键 Edge 可回溯 symbol/file/condition/state transition。

## 4.1 Control Flow（控制流）

```
transect()                     api.py:transect()
  → _run()                     api.py:_run()
    → validate_section_order() report/render.py（typo 节名前置失败）
    → validate_layers()        layers.py:validate_layers()
    → _openclaw_files()        api.py（.jsonl 分流 / eval+jsonl 混合拒绝）
    → _import_openclaw()       api.py（jsonl → snapshot DB）
    → select_transcripts()     selection.py:select_transcripts()
    → _scanners()              api.py（4 结构 + 条件 judged）
    → scout_scan(...)          scout API（fresh scan）
    → load() → _load_scan()    api.py（结果表 → TransectResults + layer_frames + scan_status）
    → render_report()          report/render.py（SECTION_KEYS 7 节 + reliability audit）
    → _spawn_viewer()          api.py（scout view 子进程；TTY hold / 非 TTY detach）
  → atexit 清理
```
关键分支：`judge_models is None` → 跳过 judged 扫描器并 WARNING（结构层独立可运行）；`epochs="all"` → 每 epoch 独立报告文件。

## 4.2 State Flow（状态流）

```
scan store: scans/scan_id=*/（Scout 约定目录）
  ├─ *_results.parquet / 结果表（token_timeline, flushes, interventions, phase_turns, ...）
  └─ _scan.json（scan 元数据：scanner 名单/transcript_ids）
        │ load() 重读（不重扫）
        ▼
TransectResults（dataclass）: token_timeline/phases/phase_turns/turn_groups/subagents/... + layer_frames + scan_status
        ▼
report.html / report_epochN.html + viewer_url（Scout 深链）
```
判定状态流：digest → (judge rows) basis=judged / filled（置信 0.3）/ attributed / unattributed / refusal / no_answer / missing_turn → vote consensus（modal 胜出）→ verifier 改写（relabel）或 weak_relabel（不应用）→ narrator 叙述（不改标签）。

## 4.3 Data Flow（数据流）

```
Inspect .eval / OpenClaw .jsonl
  → transcript（Inspect Scout 模型）
  → turn_digests()（text/reasoning/delegations snippet 截断；content-free 无 digest）  phases.py:turn_digests()
  → chunk digests（chunk=40）
  → call_judge() → 每 chunk 判定行（phase 边界/标签/置信/解释 + last_phase 携带）  phases_common.py:call_judge()
  → stitch_phases()/resolve_phases()（跨 chunk 拼接 + gap 继承）  phases_common.py
  → vote_turns()（modal → tie conf → tie order）  phases_cohort.py:vote_turns()
  → verify_phases()（4 触发选定 + 0.6 gate 改写）  phases_verify.py
  → narrate_phases()（headline/summary/turn groups）  phases_narrate.py
  → project_phase_turns()（turn 轴）  phases.py:project_phase_turns()
  → frames（20）→ report.html
```
子代理数据流：span_begin/span_end 事件（structure）→ span 活动折叠为 delegation 行锚定到 span_begin 前最近 eligible turn → subagent_classification 标签子代理回合 → subagents/subagent_votes 帧。

## 4.4 Evidence Flow（证据流）

```
transcript 事件（model/span_begin/span_end/tool 消息）        base.py:token_timeline
  → event uuid（每个 digest 携带）                            phases.py:turn_digests
  → anchor_of = {turn: event_id} → phase.anchor_event_id      phases.py:decision_phases
  → Scout viewer 深链（anchor 直达源头回合）
  → 帧行保留 transcript_id/turn/basis/label_source/confidence/agreement/judge-identity 列
  → reliability audit 从帧读取（detect_regime 读 stamped 列，非启发式）
```
关键：证据流与判定流解耦——judge 输出只落结果表；渲染时才把标签+证据并排。OpenClaw 源无 event uuid → anchor_event_id=None（溯源断裂点显式化）。

## 4.5 Authority Flow（权威流）

```
judge roster（judge_models/k_rolls 指定）
  → regime 判定（solo/k_roll/cohort）                        phases.py:decision_phases / reliability.py:Regime
  → solo/k_roll: judge 直接权威（k_roll 经投票收敛）
  → cohort: 多数投票 = 校正机制，verifier 强制 off          phases.py resolve_verify
  → verifier（触发集）: confidence>=0.6 可改写标签（relabel）否则 weak_relabel 不应用
  → narrator: 永不改标签（权威=仅叙述）                    phases_narrate.py
  → reliability audit: 最终权威层（flag 可标红，但不改数据）
```
权威分层不变量：narrator 无改判权；verifier 有条件的改判权（gate 0.6）；majority 在 cohort 中优先级高于单 judge。

## 4.6 Memory Flow（记忆流）

```
inspect-ai generate cache（judge 调用去重；cached calls 不计 token）  phases.py decision_phases cache=True / 默认
  → k_roll roll 间复用（同一模型重复 roll 的缓存行）——高自洽可能含重放（flag 预警依据）
  → scan store 持久化：load() 无扫描重读（一次扫描多次渲染/分析）
  → OpenClaw snapshot DB（uuid4 快照目录，防递归）
```
cache 语义测试锚点：`test_token_accounting.py`（normalized_input 排除 cache；later cache 不能改变 earlier derived values）；`test_kroll_self_consistency_flags_low_and_suspicious_high`。

## 4.7 Policy Flow（治理流）

```
Spec 契约（extra="forbid"） → 配置错误前置失败                  spec.py
  → 判定策略常量（LOWCONF=0.6 / _SPOT_DEFAULT=0.05 / _SAMPLE_MIN=3 / AGREEMENT_RELIABLE / AGREEMENT_TENTATIVE / KROLL_SUSPICIOUSLY_HIGH / _CI_MIN_N=8）  cohort.py / reliability.py / report/reliability.py
  → flag 策略（无阈值硬规则：spot-check overturn 必 red；unjudged 至少 amber）  report/reliability.py:build_flags
  → sensitivity 标记 → 报告首尾横幅                               api.py → render_report(sensitivity=...)
  → 治理可显式降级：verify=False / judge_models=None（成本-可信权衡开放给用户）
  → 执行 vs 判定质量分层：scan_status 只记执行，audit 只判质量     scan_status.py 头部 docstring
```
Policy Flow 闭环：Decision（阈值常量）→ Approval（前置验证拒绝）→ Policy（flag 策略）→ Enforcement（渲染横幅 + audit flags）→ Future Decision（remediation 文案反馈给用户）。

## Flow 交叉校验（Flow→KO）

| 流 | KO 挂接 |
|---|---|
| Control/Data | KO-01（判定链闭环） |
| Evidence | KO-02（溯源不变量） |
| Authority | KO-03（三层校正模式） |
| Memory | KO-04（cache 重放→高自洽设防） |
| Policy/State | KO-05（策略常量治理） |
