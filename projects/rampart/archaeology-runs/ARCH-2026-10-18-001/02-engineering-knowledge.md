# 02 · Engineering Knowledge 层（EK Graph）— RAMPART refresh（ARCH-2026-10-18-001）

> 宽底座原则：EK 是推理原材料。**本文件为 refresh 增量 EK（EK-R01~R15）**，聚焦 09-04 后新增机制；09-04 基线 28 条 EK（EK-01~28，见 corpus projects/rampart/02_engineering-knowledge.md commit 125595c）经抽查确认未变，不在此重复。
> 每条 EK 声明 links（6 类边）。证据格式：文件:行 或 文件+符号，全部可回溯 HEAD 0fb9cc2。

## EK-R01 · 双视图隔离：turns（带标注）vs raw_turns（无标注）
- **层**: L2
- **内容**: `TraceRun.turns` 是 driver/report 视图，可带 online evidence（eval_result + eval_purpose）；`TraceRun.raw_turns` 是 evaluator 视图，**永不带框架产生的评估标注**。evaluator 只看 raw_turns，driver 看 turns——判定者与被判定历史隔离。
- **证据**: rampart/core/trace.py:51-59（"turns is the driver/report view and may carry online evidence. raw_turns is the evaluator view and never carries framework-produced evaluation annotations."）
- **links**: mechanism↔EK-R02（都是评测证据分层）；causal→EK-R04（raw_turns 是 final-trace 评测输入）；dependency→EK-01（observability 仍是上下文必填）

## EK-R02 · 评测证据分层：online stop-check ≠ final-trace verdict
- **层**: L2
- **内容**: stop_when 的在线评估（`EvaluationPurpose.STOP_CHECK`）是**执行期证据**，`Result.final_trace_evaluation` 是**终局判定证据**。`trace_end_reason` 记录为什么停止产生 turn；代码**从不**从 status/turns 推断终局评测或 stop reason——"the codec never infers terminal evidence or a stop reason from the result status or turns"。
- **证据**: docs/usage/results-and-reporting.md:50-72（"final_trace_evaluation is the evaluator output for the trace when execution stops, not simply the last online evaluation"）；docs/concepts/trace-schema.md:62-65
- **links**: causal→EK-R03（分层证据通过序列化持久化）；mechanism↔EK-R01；constraint→EK-R08（v2 schema 定义 absence 语义）

## EK-R03 · 版本化 ResultRecord 信封 + decoder registry
- **层**: L2
- **内容**: `serialize_record()` 输出 `{"version": "rampart.trace.v2", "result": <body>, ...}`；`deserialize_record()` 按 version 查 `_DECODERS` registry，未知版本抛 `UnsupportedSchemaVersionError`（SchemaError 子类）。nested result body 本身**无版本**，不得单独持久化。
- **证据**: rampart/core/serialization.py:51-113（ResultRecord/VERSION/json_schema）、180-190（version 查找 + UnsupportedSchemaVersionError）、233（_DECODERS 注册 v2）；docs/concepts/trace-schema.md:12-20（"The nested result body is unversioned and must not be persisted on its own"）
- **links**: dependency→EK-R09（decoder 依赖 adapter-local schema）；causal→EK-R07（compatibility gate 守卫版本演化）

## EK-R04 · final-trace 评测复用：evaluator 身份 + 上下文匹配才复用
- **层**: L2
- **内容**: `evaluate_final_trace_async` 先检查 `run.latest_stop_check`：仅当 (1) 同一 evaluator 对象（eq=False，身份即边界）(2) `_matches_final_trace_context`（observability/manifest/raw_turns 逐项相等）时，复用在线 stop-check 结果（evidence 转 safe_str_list）；否则重新评估。避免重复 LLM 调用。
- **证据**: rampart/core/trace.py:28-38（EvaluationRecord "Identity is the reuse boundary"）、95-111（_matches_final_trace_context）、155-230（evaluate_final_trace_async）
- **links**: causal→EK-R02；dependency→EK-R01（复用基于 raw_turns 快照）；contrast↔EK-R05（复用 vs 重建是同一问题的两种策略）

## EK-R05 · 渐进迁移：final_trace_evaluation 是可选字段，status 仍权威
- **层**: L2
- **内容**: `Result.final_trace_evaluation`/`trace_end_reason` 默认 None；**XPIA 和 SingleTurn 尚未迁移**到 trace runner（_xpia.py 保留自建循环）。文档明示："execution policy can still adjust the verdict, and result.status remains authoritative"。迁移是渐进的，不是一次性替换。
- **证据**: rampart/core/result.py:162-187（字段 docstring "None for manual/error results and execution strategies that have not migrated"）；rampart/attacks/_xpia.py:100-160（自建 _run_phases_async，无 run_trace_async 调用）；docs/usage/results-and-reporting.md:66-69（"Existing prefix-evaluated strategies leave these fields as None until their follow-up migration"）
- **links**: contrast↔EK-R04（迁移态 vs 目标态）；constraint→EK-R02（未迁移时不得伪造终局证据）；mechanism↔EK-08（BaseExecution 骨架仍是唯一入口）

## EK-R06 · population provenance：id/index/size/threshold 全约束 + 交叉不变量
- **层**: L2
- **内容**: `PopulationRef.__post_init__` 校验：id 非空字符串、index 0<=index<size、size ≥1、threshold finite∈[0,1]；adapter 侧再加 cross-field 不变量 `index < size`（_validate_population）。**类型纪律显式拒绝 bool**（bool 是 int 子类陷阱，`type(value) is not int` 精确判断）。
- **证据**: rampart/core/_population.py:14-118（validate_population_id/size/threshold/index/parameters）；rampart/core/result.py:119-131；rampart/core/_schema.py:215-240（_population_fields/_validate_population）
- **links**: causal→EK-R07（v2 要求非空 id 触发 new-major）；mechanism↔EK-R09（schema 层施加 bounds，不改变 live annotations）

## EK-R07 · 版本化契约 = 显式决策对象（compatibility gate）
- **层**: L2
- **内容**: `schemas/trace-compatibility.json` 记录 version / contract_sha256 / previous_contract_sha256 / decision / rationale / migration_note。`check_trace_compatibility.py` 计算 5 个 SOURCES 文件的契约指纹，强制：首次→INITIAL（无 previous）；同 major→COMPATIBLE；major+1→NEW_MAJOR。指纹链断裂即失败。**契约变更必须先决策，后实现**。
- **证据**: schemas/trace-compatibility.json（decision="new-major"，previous_sha 链）；scripts/check_trace_compatibility.py:80-170（_base_files/_validate_decision）；tests/scripts/test_trace_compatibility.py
- **links**: constraint→EK-R03（信封版本由 gate 守卫）；causal→EK-R06（非空 id 需求触发 new-major）；mechanism↔EK-R11（都是"契约即治理对象"）

## EK-R08 · 无 upcaster 哲学：拒绝静默迁移，禁止 relabel
- **层**: L2
- **内容**: v1 schema 原样保留；**无 v1 decoder/upcaster/migration utility**。迁移规则：把 v1 记录显式转成独立 v2 记录再 `deserialize_record()`，**不得仅改版本号**；空 population id 需要合法 producer provenance 或再生成，不发明 id；未记录的 final_trace_evaluation/trace_end_reason/eval_purpose 保持 absent/null，不从在线评估推断。
- **证据**: schemas/trace-compatibility.json migration_note（"no v1 decoder, upcaster, or migration utility is provided... do not merely relabel their version"）
- **links**: contrast↔EK-R10（严格显式 vs 隐式容错）；dependency→EK-R07（决策文件承载该哲学）；constraint→EK-R02（absence 语义受此约束）

## EK-R09 · adapter-local schema：策略施加于 schema 副本，不动 live classes
- **层**: L2
- **内容**: `_schema.py` 的 `trace_schema` 通过 pydantic `GetPydanticSchema` 在序列化 adapter 上递归复制 schema 节点并施加 trace 策略（strict、allow_inf_nan=False、revalidate_instances=always、Unicode scalar、ISO datetime、JSON domain、Payload 约束、PopulationRef bounds）。**live dataclass 构造不受影响**——运行时允许的构造在序列化时可能被拒。
- **证据**: rampart/core/_schema.py:18-38（trace_schema "Apply canonical policies to an adapter's schema, not its live classes"）、39-140（_trace_schema 递归复制）、215-275（_population_fields/_trace_payload）；rampart/core/serialization.py:240-270（_result_adapter 缓存 TypeAdapter[Result]）
- **links**: dependency→EK-R03（序列化正确性依赖此层）；causal→EK-R06（population bounds 在此施加）；contrast↔EK-R05（schema 分离 vs 类内校验）

## EK-R10 · strict JSON-mode 校验：no coercion + JSON domain 递归
- **层**: L2
- **内容**: 反序列化路径：`json_value` 递归校验 JSON domain（拒绝循环/非有限/非字符串键/代理对 surrogate）；`validate_json(strict=True)` 禁止 string/bool/int coercion；`allow_nan=False` + `parse_constant` 拒绝 JSON 非有限常量；datetime 用 `fromisoformat`（Python ISO，非 RFC3339 严格）。null 仅 nullable 字段。
- **证据**: rampart/core/_schema.py:280-330（json_value/_json_value/_iso_datetime）；rampart/core/serialization.py:195-225（_validate_body）；docs/concepts/trace-schema.md:28-33（"validates nested fields without string, boolean, or integer coercion"）
- **links**: mechanism↔EK-R06（同为类型纪律）；causal→EK-R08（严格校验支撑无 upcaster 迁移）；dependency→EK-25（xdist 信任边界的同类纵深防御，见 09-04 EK）

## EK-R11 · xdist 权威归属：controller 覆盖 nodeid/index，不信任 worker
- **层**: L2
- **内容**: xdist 反序列化（SCHEMA_VERSION="rampart.xdist.v2"）强制 schema/枚举校验；**`metadata["_pytest_nodeid"]`/`_rampart_result_index` 由 controller 从 envelope nodeid 与 list position 权威设置**——"cross-worker ordering is total and independent of any (untrusted) serialized values"。worker payload 视为 attacker-controlled（头注释）。
- **证据**: rampart/pytest_plugin/_xdist.py:1-20（信任边界头注释）、58（SCHEMA_VERSION）、deserialize_report_data（"set authoritatively from the envelope nodeid and list position"）
- **links**: mechanism↔EK-25（09-04 xdist 信任边界延续）；causal→EK-R03（envelope 校验支撑）；constraint→EK-R06（population 溯源在 worker 侧重放）

## EK-R12 · duplicate timestamp 报告保真（#179）
- **层**: L1/L2
- **内容**: json_file sink 文件名 `run_report_<UTC毫秒>.json`；若已存在同文件，追加 `_<uuid4.hex>` 后缀——**不覆盖、不合并**，保证每条报告唯一落盘。
- **证据**: rampart/reporting/json_file.py:67-75（timestamp + uuid 后缀 fallback）
- **links**: mechanism↔EK-26（报告保真/不丢证据族）；causal→EK-R02（终局证据持久化落点）

## EK-R13 · RAMPARTner onboarding skill：把"首个安全测试"固化为对话契约
- **层**: L2
- **内容**: `rampart/skills/rampartner/SKILL.md` 是 Agent Skills 格式的自包含 skill（license MIT、随包分发）。契约要点：①首次回复必须以 "Howdy! I'm your RAMPARTner" 开场且只一次 ②**source choice 纪律**——build 前显式询问 main vs latest release，不推断、记录并 pin（SHA/version）③**evaluation contract 先于实现**——实现 evaluator 前先商定"测试要证明什么"（"A broad goal or permission to 'run it' does not settle SAFE"）④teach-with-every-response ⑤区分 local extension vs framework feature request。
- **证据**: rampart/skills/rampartner/SKILL.md:1-60（conversation contract/source choice/evaluation contract 段）；docs/getting-started/onboarding-skill.md
- **links**: causal→EK-R14（skill 是框架对外心智入口）；contrast↔EK-20（09-04 无此入口，documents 是唯一引导）；mechanism↔webwright 模式（同属嵌入式 self-contained skill，跨项目）

## EK-R14 · trial_config fixture 消费强制：声明与使用不可分离
- **层**: L1/L2
- **内容**: `pytest_collection_modifyitems` 校验：有 `@pytest.mark.trial` 的 item 必须消费 `trial_config` fixture（`"trial_config" not in fixturenames → UsageError`）——防止声明了 population 却不使用（否则 override 失效/门禁空转）。marker 参数白名单 {n, threshold}、拒绝 position+keyword 重复、n 正整非 bool。
- **证据**: rampart/pytest_plugin/plugin.py:212-230（_validate_trial_declarations）；rampart/pytest_plugin/_trial.py:60-112（resolve_trial_config 全部校验）
- **links**: constraint→EK-R06（声明校验与 provenance 校验同源）；causal→EK-R13（onboarding skill 教用户正确声明）

## EK-R15 · CLI override 语义：--rampart-trials 覆盖 n，不覆盖 threshold
- **层**: L1
- **内容**: `--rampart-trials N`（parse_positive_int，argparse 类型错误即拒）只覆盖 `TrialConfig.n`；threshold 永远来自 marker 声明。设计意图：CI 批量跑可放大 population，但不允许静默放松阈值。
- **证据**: rampart/pytest_plugin/_trial.py:28-36（parse_positive_int）、90-112（override 仅 n）；plugin.py:141-146（option 定义）
- **links**: constraint→EK-R06（阈值受声明约束）；causal→EK-R14（声明校验保证 override 有对象）

---
## EK Graph 边汇总（refresh 增量子图）
```
EK-R01 ─mechanism─ EK-R02 ─causal→ EK-R03 ─causal→ EK-R07 ─constraint→ EK-R08
  │ dependency                                        │dependency
  ▼                                                    ▼
EK-R04 ─causal→ EK-R05（渐进迁移）               EK-R09（adapter-local schema）──dependency→ EK-R10
  │ mechanism↔EK-R04/05 复用vs重建                        │causal
  ▼                                                      ▼
EK-R06（population provenance）──causal→ EK-R07   EK-R10 ─dependency→ EK-25（xdist 纵深，09-04）
  │ constraint                                       │mechanism
  ▼                                                  ▼
EK-R14（fixture 消费强制）──causal→ EK-R13      EK-R11（xdist 权威归属）──causal→ EK-R12（报告保真）
```
- refresh 子图 15 条 EK，出边密度：全部 ≥1（最高 EK-R02 5 条）
- 与 09-04 基线连接：EK-R01→EK-01、EK-R02→EK-08、EK-R10→EK-25、EK-R12→EK-26、EK-R13→EK-20——refresh 不是孤立新增，锚定在既有底座上

---
## Reconciliation 补充（阶段⑤ Auditor 修正，2026-10-18）

## EK-R16 · eval 标注一致性不变量：eval_purpose 必须有 eval_result（Auditor F7）
- **层**: L1/L2
- **内容**: `Turn.__post_init__` 校验 `eval_purpose is not None and eval_result is None → ValueError`——评估目的标注不能悬空（有目的必有结果）。与 EK-R02（STOP_CHECK 是执行期证据）构成一致性闭环：每条标注都能回溯到实际评估。
- **证据**: rampart/core/types.py:318-325
- **links**: constraint→EK-R02（标注必须有结果支撑）；mechanism↔EK-R05（同属类型纪律族，KO-R05 新增实例）

## EK-R17 · population size 的 allow_empty 双路径（Auditor F9）
- **层**: L1/L2
- **内容**: `validate_population_parameters(allow_empty=True)` 仅在 `result.py:264`（PopulationResult 聚合场景）使用——聚合时允许空 population（size=0 合法）；execution 路径（`execute_trials_async` → `_validate_trial_parameters`）强制 size≥1。同一条校验函数以参数化开关区分"声明执行"与"聚合读取"两种语义。
- **证据**: rampart/core/_population.py:108-124（allow_empty → allow_zero）；rampart/core/result.py:264；rampart/core/execution.py:407-428（_validate_trial_parameters 无 allow_empty）
- **links**: contrast↔EK-R06（同族校验不同开关）；constraint→EK-R06（provenance 在聚合端放宽为空）

## Reconciliation 边更新
- EK-R16：constraint→EK-R02（标注-结果闭环）；mechanism↔EK-R05
- EK-R17：contrast↔EK-R06；constraint→EK-R06
