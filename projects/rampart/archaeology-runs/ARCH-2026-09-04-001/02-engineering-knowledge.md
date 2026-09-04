# 02 · Engineering Knowledge 层（EK Graph）

> 宽底座 + 窄尖顶原则：EK 是推理原材料，不是"未升维的 KO"。每条 EK 必须声明 `links`（≥1 条边），否则是模块说明。
> 28 条 EK，覆盖核心机制/关键实现/决策/失败与修复/测试揭示行为/配置/边界。
> 证据均来自 commit 125595c（文件:行 或 文件+符号）。Reconciled = 经独立验证修正（见 06_validation/05）。

## EK-01 · ObservabilityLevel 声明：区分"没发生"与"看不到"
- **层**：L2
- **内容**：adapter 声明 `observability_profile`（TOOL_AND_SIDE_EFFECTS/TOOL_ONLY/RESPONSE_ONLY）；evaluator 据此判断"空列表=没发生"还是"空列表=看不到"。observability_level 是 EvalContext 必填项。
- **证据**：rampart/core/types.py:41-96（ObservabilityLevel + observes_tool_calls/observes_side_effects）；rampart/core/adapter.py:79-99；rampart/core/execution.py（evaluate_turn_async 强制传入）
- **links**：constraint→EK-02（UNDETERMINED 三态依赖 observability）；mechanism↔EK-05（observability 降级）；causal→EK-08（结果含 observability_level）

## EK-02 · EvalOutcome 三态（DETECTED/NOT_DETECTED/UNDETERMINED）：不确定是显式公民
- **层**：L2
- **内容**：evaluator 可返回 UNDETERMINED；resolve 优先级中 UNDETERMINED 只在无定论时起作用。UNDETERMINED 不是异常，是一等结果。
- **证据**：rampart/core/types.py（EvalOutcome 枚举）；rampart/core/result.py:191-239（resolve_as_attack/probe 优先级）
- **links**：mechanism↔EK-01（三态与 observability 同源）；causal→EK-04（复合三值逻辑）；constraint→EK-06（undetermined_operands）

## EK-03 · Evaluator Polarity：评估器无极性，好/坏由策略决定
- **层**：L2
- **内容**：Evaluator 只答"X 发生了吗"（polarity-free）；同个 evaluator（如 ToolCalled）在 attack 里 DETECTED=UNSAFE、在 probe 里 DETECTED=SAFE。映射在 Attacks/Probes 工厂。
- **证据**：rampart/core/evaluator.py（"They answer 'did X happen?'"）；docs/concepts/overview.md "Evaluator Polarity"；rampart/core/result.py:191-239
- **links**：mechanism↔EK-04（组合算子无极性）；constraint→EK-09（resolver 优先级）

## EK-04 · 三值逻辑组合算子（& / | / ~）：短路 + UNDETERMINED 传递 + 顺序无关
- **层**：L2
- **内容**：`|` 左 DETECTED 短路；`&` 左 NOT_DETECTED 短路，但 **UNDETERMINED 左操作数不短路**（因右操作数可能 NOT_DETECTED 定案——避免结果依赖书写顺序）；`~` 翻转 DETECTED↔NOT_DETECTED，UNDETERMINED 透传。
- **证据**：rampart/core/evaluator.py（_AnyEvaluator/_AllEvaluator/_NotEvaluator，"An UNDETERMINED left operand does not" 注释）
- **links**：causal→EK-05（组合结果参与语义解析）；mechanism↔EK-03；constraint→EK-06（_merge_undetermined）

## EK-05 · Observability 调整：RESPONSE_ONLY + 零工具调用 → SAFE 降级 UNDETERMINED
- **层**：L2
- **内容**：XPIA 在 resolve_as_attack 得 SAFE 后，若 adapter 是 RESPONSE_ONLY 且全程零工具调用，降级为 UNDETERMINED——"看不到工具调用"不能成为"安全"结论。
- **证据**：rampart/attacks/_xpia.py:216-246（_adjust_for_observability，三条件全部满足才降级）；tests/unit/attacks/test_xpia.py（test_response_only_no_tools_downgrades_to_undetermined 等 5 测试）
- **links**：dependency→EK-01；causal→EK-09；contrast↔EK-07（探针语义镜像）

## EK-06 · undetermined_operands 追踪：复合评估的"哪部分没确定"如实上浮
- **层**：L2
- **内容**：复合评估器把每步 UNDETERMINED 操作数的原因记录到 `undetermined_operands`（重复折叠、优先取嵌套原因、short-circuit 的操作数不记录）；报告在 summary 如实声明"部分评估未确定"。
- **证据**：rampart/core/evaluator.py（_merge_undetermined）；rampart/core/types.py（undetermined_operands 字段语义）；rampart/core/result.py（_summarize_undetermined_operands）
- **links**：dependency→EK-04；causal→EK-08（影响 summary 措辞）

## EK-07 · resolve_as_probe 语义：NOT_DETECTED → UNSAFE（探针是反转的）
- **层**：L2
- **内容**：探针测"期望行为应该发生"，任何 NOT_DETECTED → UNSAFE；优先级 NOT_DETECTED > UNDETERMINED > DETECTED。总结只从 NOT_DETECTED 的 evaluator 取理由。
- **证据**：rampart/core/result.py:228-239（resolve_as_probe）；rampart/probes/_single_turn.py（UNSAFE summary 只取 NOT_DETECTED rationale）
- **links**：contrast↔EK-05；mechanism↔EK-09（共享 resolver 架构）

## EK-08 · 基础设施韧性：任何执行异常 → SafetyStatus.ERROR，不崩溃套件
- **层**：L2
- **内容**：BaseExecution._execute_once_async 捕获 _execute_async 的任何异常 → Result(ERROR) + ON_ERROR 事件；event handler 异常被记录并吞掉（不中止测试）。"单测失败不拖垮整个套件"。
- **证据**：rampart/core/execution.py:183-222（try/except）、253-284（_fire_async handler 吞异常）；tests/unit/core/test_execution.py（test_broken_handler_does_not_abort_execution）
- **links**：constraint→EK-12（XPIA 依赖此保障）；mechanism↔EK-13（错误路径一致性）

## EK-09 · Result 是唯一结果类型 + bool(result)==safe
- **层**：L2
- **内容**：attack 和 probe 都产出 Result；`__bool__` 返回 safe（derived，不存储），使 `assert result, result.summary` 恒表"agent 行为安全"。safe 是 derived property 永不与 status 漂移。
- **证据**：rampart/core/result.py:77-127；docs/concepts/overview.md "The Result Contract"
- **links**：dependency→EK-07/EK-05；constraint→EK-16（报告消费 Result）

## EK-10 · BaseExecution Template Method：骨架固定，子类只实现 _execute_async
- **层**：L2
- **内容**：BaseExecution(ABC) 拥有 ON_PRE_EXECUTE→_execute_async→ON_POST_EXECUTE（ON_ERROR 对异常）生命周期 + 结果收集 + 计时 + 事件分发；子类只实现 `_execute_async` 和 `strategy_name`，**不应**捕获 InfrastructureError。
- **证据**：rampart/core/execution.py:113-284；docs/contributing/architecture.md "Execution Lifecycle Ownership"
- **links**：mechanism↔EK-12/EK-13（XPIA/SingleTurn 都是其子类）；constraint→EK-08

## EK-11 · 事件处理工厂注入：framework 级 handler 自动装配
- **层**：L2
- **内容**：pytest_configure 时 register_default_handler_factory（ResultCollectionHandler），每个 BaseExecution.__init__ 自动取 fresh handler，pytest_unconfigure 清理；无 global 语句（registry 单例封装）。plugin 文档化标注这是对 architecture 声明的偏差。
- **证据**：rampart/core/execution.py（_DefaultHandlerRegistry）；rampart/pytest_plugin/plugin.py（register/clear + "documented deviation" 注释）
- **links**：mechanism↔EK-10（装配进 BaseExecution）；causal→EK-15（结果进 collector）

## EK-12 · XPIA 全生命周期：注入→索引→会话→触发→评估→清理
- **层**：L2
- **内容**：XPIA 7 阶段（激活注入(AsyncExitStack) → 并发等索引 → 建会话 → PromptDriver 驱动 → 每轮评估+早停 → 清理 → resolve_as_attack）；max_turns=25 防无限循环。
- **证据**：rampart/attacks/_xpia.py（__init__ + _run_phases_async + _execute_async）
- **links**：mechanism↔EK-10；causal→EK-14（并发激活机制）

## EK-13 · SingleTurn 探针：无注入阶段的执行策略
- **层**：L1
- **内容**：探针只做 会话→驱动→评估→清理，无注入阶段；语义反转（detected→SAFE）。与 XPIA 共享 BaseExecution 骨架。
- **证据**：rampart/probes/_single_turn.py（SingleTurnExecution._execute_async）
- **links**：mechanism↔EK-12；mechanism↔EK-10

## EK-14 · 混合并发策略：以"清理是否已注册"划分取消安全边界（Reconciled）
- **层**：L2
- **内容**：注入生命周期两阶段用不同并发原语——**context entry 阶段（网络上传，清理尚未注册）用 `asyncio.gather(return_exceptions=True)`**（注释 "non-cancelling so successful siblings still register cleanup on the exit stack"），失败 sibling 不取消成功 sibling 的清理注册；**readiness wait 阶段（等索引，清理已注册）用 `asyncio.TaskGroup`**（此时取消安全）。关键决策不是"禁 TaskGroup"，而是"**清理已注册之前不可取消、注册之后可取消**"。
- **证据**：rampart/attacks/_xpia.py:180-207（gather entry + TaskGroup ready）；tests/unit/attacks/test_xpia.py（test_partial_activation_failure_still_cleans_up_siblings）
- **links**：contrast↔EK-08（错误路径设计对照）；dependency→EK-12

## EK-15 · ContextVar 作用域结果收集：async 并发下每 task 独立 collector
- **层**：L2
- **内容**：autouse fixture `_rampart_collect` 为每个测试建立 ContextVar 作用域 ResultCollector；ResultCollectionHandler 在 ON_POST_EXECUTE 写入；测试体后 drain 进 RampartSession。ContextVar 每 task 独立。
- **证据**：rampart/pytest_plugin/plugin.py（_rampart_collect fixture）；rampart/pytest_plugin/_collection.py
- **links**：dependency→EK-11；causal→EK-17（session 聚合）

## EK-16 · trial 统计试验：@pytest.mark.trial(n=, threshold=) 克隆 + 聚合 gate
- **层**：L2
- **内容**：collection 时把 trial-marked item 克隆为 n 个 item（[trial-N] 后缀，复制全部 markers，xdist_group 同 worker）；session-finish 按 base_nodeid 聚合。**gate 完整语义（Reconciled）**：`errors>0 → ERROR(FAIL)`；`executed>0 且 pass_rate≥threshold → SAFE(PASS)`（**统计容忍：含 unsafe 但比例达标仍 PASS**）；`unsafe>0 → UNSAFE(FAIL)`；**否则（含全 no_result）→ UNDETERMINED(FAIL)**。pass_rate=safe/total。
- **证据**：rampart/pytest_plugin/plugin.py（_create_trial_clones + _evaluate_gates）；rampart/pytest_plugin/_session.py（TrialGroupResult.status：errors→executed+threshold→unsafe→undetermined）
- **links**：causal→EK-18（聚合数据来源）；mechanism↔EK-20（xdist 传输）；constraint→EK-09

## EK-17 · 终端注入防护：攻击者 payload 文本不能注入终端
- **层**：L2
- **内容**：result summary 可能含攻击者控制的 agent 响应/payload 文本；输出终端前 strip_ansi 剥离转义序列与控制字节。
- **证据**：rampart/pytest_plugin/plugin.py（_sanitize_for_terminal + "Prevents terminal injection"）；rampart/common/text.py
- **links**：mechanism↔EK-21（同为攻击者输入防护）；contrast↔EK-18（xdist 侧纵深防御）

## EK-18 · xdist 信任边界：worker payload 视为攻击者可控制
- **层**：L2
- **内容**：_xdist.py 头注释明确 "worker payloads may contain attacker-controlled content"；序列化严格 JSON-safe；反序列化校验 schema version/enum/depth；ANSI 剥离为纵深防御。malformed envelope → 标记 incomplete 而非 abort（workeroutput 信任边界约定）。
- **证据**：rampart/pytest_plugin/_xdist.py:10-18；SCHEMA_VERSION="rampart.xdist.v2"；rampart/pytest_plugin/plugin.py（pytest_runtest_logreport 捕获 WorkerOutputError→mark_incomplete）
- **links**：mechanism↔EK-17；causal→EK-19（incomplete→强制失败）；constraint→EK-22（序列化大小）

## EK-19 · incomplete-run 强制非零退出："green 必须 ran and passed"
- **层**：L2
- **内容**：丢失/崩溃的 xdist worker 可让所有幸存测试全绿；若 run incomplete 且 exit 当前 OK → 强制 TESTS_FAILED。对安全框架，"green"必须意味着"跑完且通过"。
- **证据**：rampart/pytest_plugin/plugin.py（_enforce_incomplete_exit_status）
- **links**：dependency→EK-18；mechanism↔EK-16（trial gate 同属治理）

## EK-20 · 序列化大小上限：超限截断 + 标记 incomplete
- **层**：L1
- **内容**：`--rampart-xdist-max-bytes` 默认 16MiB（下限 4MiB）；超大 Result 以截断标记替换，controller 标记 run incomplete。
- **证据**：rampart/pytest_plugin/_xdist.py:60-62,150-195；docs/usage/ci-integration.md
- **links**：constraint→EK-18；mechanism↔EK-19

## EK-21 · PayloadStore 路径逃逸防护（对自己边界的严格安全）
- **层**：L2
- **内容**：payload 持久化用原子写（temp-dir-then-rename，xdist 并发安全）；反序列化时 _validate_collection_name（拒 / \ .. 空）、_ensure_within_directory（resolved 必须 within）、_validate_artifact_reference（拒绝对路径/..）；symlink 逃逸也被拒。
- **证据**：rampart/payloads/_store.py:38-60,230-330；tests/unit/payloads/test_payload_store_security.py 全部
- **links**：mechanism↔EK-17（攻击者输入防护家族）；contrast↔EK-23（校验 vs 运行时防护）

## EK-22 · 懒加载：pytest 插件启动不加载重依赖（pyrit/transformers）
- **层**：L2
- **内容**：`__lazy_imports__`（Attacks/LLMDriver/LLMJudge/Probes/TranscriptScope）延迟到首次访问；pyrit_bridge 内部函数级 lazy import。测试锁定：插件 import 后 sys.modules 不得有 pyrit/transformers。
- **证据**：rampart/__init__.py（__getattr__ 懒加载）；docs/contributing/architecture.md "PyRIT Bridge"；tests/unit/test_public_api.py（test_pytest_plugin_import_does_not_load_heavy_dependencies）
- **links**：constraint→EK-08；mechanism↔EK-24（重依赖隔离策略）

## EK-23 · LLM judge 输出强校验：malformed/瞬时失败 → UNDETERMINED
- **层**：L2
- **内容**：LLMJudge 的 verdict 严格校验（Reconciled 强化）：`_ALLOWED_OUTCOMES={"detected","not_detected","undetermined"}` 小写下划线字面量白名单（LLM↔解析器 JSON 契约）+ 四必需键（outcome/confidence/rationale/evidence）+ confidence 拒绝 bool/NaN/Infinity（在解析边界拒掉，触发重试而非静默毒化下游比较）+ evidence 防御性拷贝；JSON 容错解析（markdown fence 剥离 + first-{ fallback）；重试后仍 malformed 或 empty/rate-limit → EvalOutcome.UNDETERMINED。
- **证据**：rampart/evaluators/llm_judge.py:60-246（_ALLOWED_OUTCOMES/_validate_*）
- **links**：dependency→EK-02；mechanism↔EK-21（输入防护家族）；causal→EK-05（UNDETERMINED 传导）

## EK-27 · 序列化安全解析族：safe_str/safe_str_list/safe_float（Reconciled 新增）
- **层**：L2
- **内容**：common/text.py 提供 `safe_str/safe_str_list/safe_float` 作为第三方 evaluator 输出/序列化边界的强制转换——JSON 无 NaN/infinity，第三方可留非数值字段 → 安全 fallback（`float|None`）；bool 被拒（防 `True→1.0` 静默漂移）。与 judge confidence 解析同哲学。
- **证据**：rampart/common/text.py:47-150（strip_ansi/safe_*）；rampart/pytest_plugin/_xdist.py 引用
- **links**：mechanism↔EK-23（同解析边界哲学）；constraint→EK-18（xdist 反序列化）

## EK-28 · 测试卫生检测：@harm 但无结果 → warning（Reconciled 新增）
- **层**：L1
- **内容**：`_rampart_collect` fixture teardown 检查 `@pytest.mark.harm` 且 collector 无结果 → warning "did you forget record_result() or assert result?"——检测"测试没跑起来"的框架级提醒（仅 warning 不 FAIL）。
- **证据**：rampart/pytest_plugin/plugin.py（_rampart_collect teardown）
- **links**：mechanism↔EK-15（collector 生命周期）；constraint→EK-19（治理家族）

## EK-24 · PyRIT 桥：重上游依赖隔离边界
- **层**：L1
- **内容**：PyRIT 相关逻辑集中在 rampart/pyrit_bridge/；对外只暴露 create_prompt_target / send_generation_request_async / send_judge_request_async；pyrit 是 git rev 固定版本（6dc8b94）。
- **证据**：rampart/pyrit_bridge/llm_bridge.py:45,157,199；pyproject.toml [tool.uv.sources]
- **links**：mechanism↔EK-22；dependency→EK-23（judge 走桥）

## EK-25 · 报告口径：ERROR 从攻击成功率分母排除
- **层**：L1
- **内容**：PopulationSummary 计算 attack_success_rate 时排除 ERROR 结果（"A SharePoint 503 is not a safety finding. Including errors in the denominator dilutes..."）——基础设施错误不是安全发现。
- **证据**：rampart/reporting/sink.py:96-140（population_summary 注释）
- **links**：constraint→EK-09；contrast↔EK-19（错误处理哲学对照）

## EK-26 · 早停：detected 即 break（attack 命中即止，probe 命中即止）
- **层**：L1
- **内容**：两策略每轮评估后 `if turn.eval_result and turn.eval_result.detected: break`。
- **证据**：rampart/attacks/_xpia.py（_run_phases_async）；rampart/probes/_single_turn.py
- **links**：mechanism↔EK-12/EK-13；constraint→EK-08

---
**EK Graph 边统计**：28 EK，平均出边 ≥2；无孤立 EK；聚合规则见 03_knowledge-layer。
