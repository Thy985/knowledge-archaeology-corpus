# 02 Engineering Knowledge — Transect（ARCH-2026-10-13-001）

> EK Graph：15 条 Engineering Knowledge，六类边（mechanism/subsystem/causal/dependency/constraint/contrast）。每条保留证据来源（file:symbol）。孤立即降 D 级——本表 0 游离。

## EK Graph 总览（六类边）

| 边 | 连接 |
|---|---|
| mechanism | EK-03↔EK-04（投票共识机制）；EK-02↔EK-16（digest 溯源）；EK-05↔EK-06（verifier/narrator 均为二轮 LLM 面） |
| subsystem | EK-01↔EK-02↔EK-07↔EK-10（scanners 层）；EK-08↔EK-13↔EK-14（frames/reliability/scan_status 判定与执行数据面）；EK-09↔EK-15（分发与扩展面） |
| causal | EK-02→EK-03→EK-04→EK-05→EK-06（digest→judge→vote→verify→narrate 判定链）；EK-07→EK-08（reliability 模型→flag 策略）；EK-01→EK-10（结构扫描→选择约束） |
| dependency | EK-03 依赖 EK-02（judge 输入依赖 digest）；EK-08 依赖 EK-07（flag 依赖统计模型）；EK-16 依赖 EK-02（溯源锚点依赖 digest event_id） |
| constraint | EK-04 约束 EK-05（cohort 时 verifier 强制 off）；EK-13 约束 EK-08（执行记录与判定质量分离，flag 不掺执行）；EK-09 约束 EK-01（layer 名不得撞内建帧名） |
| contrast | EK-03 solo↔cohort（单判 vs 投票）；EK-05 verifier↔EK-06 narrator（可改写 vs 只叙述）；EK-11 structural($0)↔judged（零成本 vs 模型花费） |

## EK 条目

### EK-01 结构/判定双层扫描器架构
- **内容**：默认扫描批 = 4 个结构扫描器（token_timeline/context_flush/human_intervention/eval_setup，$0 零 LLM 成本）+ 条件化 judged 扫描器（decision_phases 需 spec.phases；subagent_classification 需 spec.subagent_labels；judge_models 为 None 时仅结构层并打 WARNING）。
- **证据**：`src/transect/api.py:_scanners()`；`scanners/base.py` 头部 docstring "Structural (zero-LLM) Scout scanners: the default path."
- **links**: subsystem EK-02, EK-07, EK-10; causal EK-01→EK-10; contrast EK-11

### EK-02 turn digest 稀疏化（判定输入压缩）
- **内容**：`turn_digests()` 每个 reasoning-bearing orchestrator turn 产一条 digest：text/reasoning/delegations 各自 `snippet_chars` 截断；content-free tool-call-only turn 无 digest；子代理 span 活动折叠为 delegation 行、锚定到 span_begin 前最近 eligible turn；OpenClaw 失败占位 turn 排除。`agent_task_prompt()` 提取首个非 scaffold 用户消息作任务上下文（TASK_PROMPT_CHARS=2400）。
- **证据**：`src/transect/scanners/phases.py:turn_digests()` / `agent_task_prompt()` / `_is_failed_turn()`；常量 `SNIPPET_CHARS`（phases_common）
- **links**: mechanism EK-16; subsystem EK-01, EK-07, EK-10; dependency EK-03→EK-02

### EK-03 三 regime 判定（solo/k_roll/cohort 互斥）
- **内容**：solo=1 模型 1 roll 单判；k_roll=1 模型多 roll 投票驱动分区；cohort=多模型逐单元多数投票。`cohort_members()` 生成成员（(name, roll) 身份键）；verify 自动规则：单模型（solo/k_roll）on、cohort off（"majority vote is the correction mechanism"）。
- **证据**：`src/transect/scanners/phases.py:decision_phases()` docstring 与 `resolve_verify()`；`scanners/cohort.py:Member.key`
- **links**: mechanism EK-04; subsystem EK-01, EK-07, EK-10; causal EK-02→EK-03; constraint EK-04→EK-05; contrast EK-03 solo↔cohort

### EK-04 投票共识规则（modal → tie mean conf → tie order）
- **内容**：`vote_ballots()`/`vote_turns()`：模态标签胜出；平局按票侧平均置信度更高者；再平局按成员顺序最早者。agreement = 模态侧票数/n_voting。solo 单元无 agreement/CI。
- **证据**：`src/transect/scanners/cohort.py:vote_ballots()`；`scanners/phases_cohort.py:vote_turns()` docstring
- **links**: mechanism EK-03; subsystem EK-01, EK-07, EK-10; constraint EK-04→EK-05

### EK-05 verifier 二轮审计（4 触发 + 0.6 overturn gate）
- **内容**：`select_for_verify()` 触发：low_confidence（min_confidence<0.6）、low_agreement（<0.6，仅 k_roll）、wedge（≤2-turn 夹在共享另一标签的两 phase 间）、random_sample（默认 max(5%,3) 固定种子分层抽样）。verifier 判定：confidence≥0.6 且标签不同 → 应用改写（relabel）；<0.6 分歧 → weak_relabel 记录不应用；无答案/拒绝 → no_verdict 计数。同模型 verifier 用第二 procedure/prompt（不度量未变程序可重复性）。
- **证据**：`src/transect/scanners/phases_verify.py:select_for_verify()` + `VerifierAudit` 字段；`cohort.py:LOWCONF=0.60`
- **links**: mechanism EK-06; subsystem EK-01, EK-07, EK-10; causal EK-02→EK-03→EK-04→EK-05→EK-06; contrast EK-05 verifier↔EK-06 narrator

### EK-06 narrator 叙述层（不改标签）
- **内容**：`narrate_phases()` 为（验证后）phase 产 headline+summary+turn groups；`narration_group_status` 描述坐标完整性（complete/invalid_partition/empty_groups/no_narrative/not_run），**非事实正确性**；narrator 失败有 fallback 计数（NarratorAudit.ran/n_fallback）。
- **证据**：`src/transect/scanners/phases_narrate.py:NarratorAudit`；`phases.py:decision_phases()` narration 字段文档
- **links**: mechanism EK-05; subsystem EK-01, EK-07, EK-10; contrast EK-06 narrator↔EK-05 verifier

### EK-07 reliability 数据模型（Regime/Rate/α/AC1/Mean）
- **内容**：`Regime`（none/solo/k_roll/cohort + n_models/k_rolls/models/verifier_on/verifier_same_model）；`Rate`（k-of-n + Wilson 95% CI，分母 0 → None）；`CohortAgreement`（percent_agreement + Krippendorff α nominal + Gwet AC1，互不为 bounds）；`Mean`（CI 半宽，N<8 不出以防水准高估）；`LabelStats`/`RelabelRate`/`MemberCoverage`（含 miss reason 计数）。
- **证据**：`src/transect/reliability.py` 全部 dataclass + `wilson_interval()`；`_CI_MIN_N = 8`
- **links**: subsystem EK-01, EK-02, EK-03, EK-04, EK-05, EK-13; causal EK-07→EK-08; dependency EK-08 依赖 EK-07

### EK-08 flag 策略（对高自洽度也设防）
- **内容**：`build_flags()` 阈值常量：k_roll 自洽均值 <0.80（AGREEMENT_RELIABLE）amber、>0.95（KROLL_SUSPICIOUSLY_HIGH）amber（"near-perfect self-agreement can reflect the cases, sampling settings, shared prompts or cache replay"）；cohort α <0.66（AGREEMENT_TENTATIVE）red / 0.66-0.80 amber；verifier 改写率 >0.20（RELABEL_RATE_HIGH）高亮；unjudged 占比 ≥0.25（UNJUDGED_SHARE_SERIOUS）→ red；random-sample spot-check overturn **必 red**；unjudged **至少 amber**。每 flag 自带 explanation + remediation（自包含）。
- **证据**：`src/transect/report/reliability.py:build_flags()` + 模块级阈值常量
- **links**: subsystem EK-13; causal EK-07→EK-08; dependency EK-08 依赖 EK-07; constraint EK-13 约束 EK-08

### EK-09 Layer 扩展机制（scanner/frame/section/tags/audit）
- **内容**：`Layer` 捆绑自定义 scanner（工厂可声明 judge 参数子集）、frame（映射或现成 DataFrame）、typed section blocks、phase-card tags、audit（(unit_col,label_col) 加入可靠性 audit 实体块）。`validate_layers()` 前置拒绝：空名/重名/撞内建帧名/无事可做/scanner+现成帧/无 registry 身份/共享 scanner/畸形 section。judged layer scanner 应建于 `cohort_llm_scanner`（自动盖章 judge 身份块 + item 元数据）。
- **证据**：`src/transect/layers.py:Layer` + `validate_layers()` docstring
- **links**: subsystem EK-15; constraint EK-09→EK-01（builtin_frame_names 防撞）

### EK-10 单样本选择规则（one-sample + epoch + task_set 拒绝）
- **内容**：`select_transcripts()`：多 sample log 必须显式 sample=（错误列出 id）；epoch auto=最早成功（无成功取最早）；同 (sample,epoch) 跨多 task_set → **拒绝而非静默保留一个**；空 index 抛。输入侧 `_openclaw_files()` 拒绝 eval+jsonl 混合；重复 transcript id 拒绝。
- **证据**：`src/transect/selection.py:select_transcripts()`；`api.py:_openclaw_files()` / `_import_openclaw()`
- **links**: subsystem EK-01, EK-02, EK-07; causal EK-01→EK-10

### EK-11 零成本结构先行 vs 模型判定分离
- **内容**：token_timeline（per-model-turn token usage + sub-agent spans + lane_activity，turn 轴）等结构扫描器在无 judge_models 时仍产完整帧；判定价在报告为可选面（WARNING 明确缺失）。成本控制 = $0 结构 + 缓存（inspect-ai generate cache 默认 True，judge 调用去重；roll_cache）。
- **证据**：`scanners/base.py:token_timeline()`；`api.py:transect()` WARNING 分支；`phases.py:decision_phases(cache=True)`
- **links**: contrast EK-01 结构 vs judged; subsystem EK-01, EK-02

### EK-12 spec 契约（extra="forbid" + ops 保留 + plain 归一）
- **内容**：`Spec`/`Phase`/`SubagentLabel` 均 `extra="forbid"`（拼错字段名 = ValidationError，杜绝静默忽略）；Phase.ops 标记 operational bucket，未声明则保留追加 "ops"；`_plain_labels_ok` 把 `"recon"` 归一为 `{"label": "recon"}`；context 仅进 phase prompt 不进 subagent judge。
- **证据**：`src/transect/spec.py` 全部
- **links**: subsystem EK-01; causal EK-12→EK-03（spec 决定 judged 面）

### EK-13 执行记录与判定质量分层（scan_status vs reliability）
- **内容**：`build_scan_status()` 只记执行级事实（哪些 scanner 跑了/多少 transcripts/记录错误/模型计费，traceback 剔除）；判定质量（弃权/填充/成员与 verifier 退化）归 reliability audit——"Scout completion describes execution, not usable labels."
- **证据**：`src/transect/scan_status.py` 头部 docstring；`ScanError`（refusal 标记，无大 traceback）
- **links**: subsystem EK-07, EK-08; constraint EK-13 约束 EK-08

### EK-14 OpenClaw telemetry hal 导入（vendored + snapshot DB）
- **内容**：OpenClaw `.jsonl` 经 vendored `openclaw_telemetry_hal` 导入 `transcript_snapshots/<uuid4>` DB（新快照放既有 DB 递归搜索根之外）；导入后从 DB 扫描；span 结束合成（ends_recorded=False）；失败占位 turn 标记；scaffold 消息精确匹配跳过。
- **证据**：`api.py:_import_openclaw()` / `_openclaw_files()`；`scanners/base.py:token_timeline()` ends_recorded 分支；`phases.py:_is_failed_turn()` / `_OPENCLAW_SCAFFOLD_USER_MESSAGES`
- **links**: subsystem EK-10（输入分流）; dependency EK-14→EK-01（导入产物进结构扫描）

### EK-15 Claude Code skills 分发（wheel 内嵌 + 幂等安装）
- **内容**：`.claude/skills/`（using-transect/transect-diagnostics/add-a-layer/custom-ui）随 wheel force-include 进 `transect/_skills`；`python -m transect.skills install` 逐 skill 幂等复制进项目 `.claude/skills/`（staged 替换失败回滚；别名跳过；升级刷新副本）。AGENTS.md/CLAUDE.md 存在。
- **证据**：`pyproject.toml` force-include 段；`src/transect/skills.py:install()`
- **links**: subsystem EK-09; mechanism EK-09（扩展分发一体）

### EK-16 标签→源头可追溯（anchor_event_id + Scout 深链）
- **内容**：每个 digest 带 event uuid（`anchor_of = {d.turn: d.event_id}`）；每个 phase 取范围内首个有 uuid 的 turn 作 `anchor_event_id`（Scout viewer 深链锚点）；帧列携带 judge-identity（judge_regime/n_models/k_rolls/verifier_armed/verifier_same_model + judge_models/verifier_model）供 reliability 读取；OpenClaw 源无 event uuid 时锚点 None。
- **证据**：`scanners/phases.py:decision_phases()` anchor 块；`frames/common.py:JUDGE_COLS/VERIFIER_COLS/judge_identity`；`reliability.py:detect_regime()`
- **links**: mechanism EK-02; dependency EK-16 依赖 EK-02; subsystem EK-07, EK-08

## 计数

- EK 总数：**16**（EK-01 ~ EK-16，连续编号）
- links 覆盖率：16/16（每条 ≥1 边）；游离 EK：0
- 六类边全用：mechanism/subsystem/causal/dependency/constraint/contrast ✓
- 平均出边：约 3.4/条
