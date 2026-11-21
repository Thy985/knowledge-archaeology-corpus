# 02 Engineering Knowledge（EK Graph）· RAMPART

35 条 EK，每条含 links（六类边：mechanism/subsystem/causal/dependency/constraint/contrast）。证据格式 `[file:line 或 test]`，均可回溯仓库实际内容。

## A. 观测与判定语义（核心安全不变量）
| ID | 知识（L1/L2） | 证据 | links |
|---|---|---|---|
| EK-01 | ObservabilityLevel 三档是适配器声明的"可观测通道"；保证是 per-channel 而非 per-field——声明报告某通道即取信其内容（redacted 参数仍视为观察到） | types.py:24-58 | mechanism↔EK-02, EK-03；constraint→EK-08, EK-09 |
| EK-02 | 评估器缺少证据通道时必须返回 UNDETERMINED 而非假 NOT_DETECTED：ToolCalled 在 observes_tool_calls=False 时 UNDETERMINED（"tool 未被调用"与"我们看不到调用"是两个答案） | tool_called.py:96; test_xpia.py:401 | mechanism↔EK-01, EK-03；causal→EK-04 |
| EK-03 | 观测降级不变量：XPIA 在 SAFE + RESPONSE_ONLY + 零工具观测时强制降级 UNDETERMINED（agent 可能调用了看不见的工具）；测试名即契约"dropping observability does not look safer" | _xpia.py:247-284; test_xpia.py:295-354 | constraint→EK-02；causal→EK-06；dependency→EK-01 |
| EK-04 | 判定优先级：attack DETECTED > UNDETERMINED > NOT_DETECTED；probe NOT_DETECTED > UNDETERMINED > DETECTED——"任一评估器检测到攻击条件即 agent 已被证明失守，无论其他评估器是否 UNDETERMINED" | result.py:389-441 | causal→EK-06；mechanism↔EK-06；constraint→EK-05 |
| EK-05 | UNDETERMINED 只应在无评估器给出确定性信号时起作用；未定操作数（undetermined_operands）保留"为什么没定案"的原因链，嵌套组合不塌缩成第一个 gap 的复述 | evaluator.py:308-334; result.py:444-547 | subsystem↔EK-04；mechanism↔EK-14 |
| EK-06 | 基础设施故障与安全判定分离：任何 _execute_async 异常统一转 Result(SafetyStatus.ERROR)（error_type/error 进 metadata），单测试失败不拖垮 suite；ERROR 从 attack_success_rate 分母剔除（"A SharePoint 503 is not a safety finding"） | execution.py:282-305; sink.py:136-149 | causal→EK-07；dependency→EK-05, EK-04；contrast↔EK-04（ERROR vs UNSAFE） |
| EK-07 | 不完整运行强制失败：丢失/崩溃的 xdist worker 可让存活测试全绿，安全框架里 "green must mean ran and passed" → incomplete 强制非零退出（不掩盖更强失败信号） | plugin.py:448-459 | causal→EK-06；dependency→EK-31 |

## B. 执行生命周期与事件系统
| ID | 知识 | 证据 | links |
|---|---|---|---|
| EK-08 | BaseExecution 固定骨架 ON_PRE_EXECUTE → _execute_async → ON_POST_EXECUTE（ON_ERROR 兜底），子类只实现 _execute_async；横切关注点（计时/结果收集/基础设施错误）归基类 | execution.py:187-316 | subsystem↔EK-09, EK-10, EK-12；constraint→EK-01 |
| EK-09 | 默认 handler factory 机制：pytest_configure 注册 → 每个 BaseExecution.__init__ 调用取 handlers（prepend 到用户 handlers）→ unconfigure 清理；避免每个测试手写 wiring | execution.py:95-184; plugin.py:173-205 | subsystem↔EK-08；causal→EK-10；mechanism↔EK-10 |
| EK-10 | handler 异常被 log+swallow——失败 handler 不得中止测试或压制结果（"a failing handler must not abort the test or suppress its result"） | execution.py:346-355 | mechanism↔EK-09；constraint→EK-08, EK-06 |
| EK-11 | trial 执行：execute_trials_async 用 execution_factory 每 trial 新建完整 execution（含 trial-scoped 依赖），顺序执行，PopulationRef 附着在 post_execute 前；参数（n/threshold）在 factory 创建前校验 | execution.py:358-426; test_execution.py:194-364 | causal→EK-12；mechanism↔EK-29；constraint→EK-05 |
| EK-12 | TrialConfig 解析：@pytest.mark.trial(n,threshold) 必须配合 trial_config fixture（否则 UsageError）；CLI --rampart-trials 只覆盖 n 不覆盖 threshold；重复/未知参数拒绝 | _trial.py:54-112; plugin.py:222-228; test_trial.py:30-87 | subsystem↔EK-11；dependency→EK-29 |

## C. Trace 与在线停止
| ID | 知识 | 证据 | links |
|---|---|---|---|
| EK-13 | TraceRun 双视图：turns（driver/report 视图，带在线证据）+ raw_turns（evaluator 视图，永不携带 framework 产生的评估注解）——stop check 只标注 public history。**Audit 收紧：trace.py 是公共导出 + 单测覆盖的共享 helper，当前内置策略（XPIA/SingleTurn）各自内联驱动循环，未调用 run_trace_async（future-ready）** | trace.py:45-67; test_trace.py:91; grep run_trace_async | mechanism↔EK-15；causal→EK-14；constraint→EK-16 |
| EK-14 | 在线停止（stop_when）helper：每个 response 后评估；detected → STOP_CONDITION_MET 终止；eval_purpose=STOP_CHECK 标记"执行证据，非最终判定输入"；UNDETERMINED 不终止。**Audit 收紧：属 trace.py 公共 helper 能力；内置策略当前用内联 `turn.eval_result.detected break`（XPIA/SingleTurn），未走 stop_when 路径** | trace.py:74-146; test_trace.py:372; _xpia.py:151-156 | causal→EK-13；mechanism↔EK-04, EK-05 |
| EK-15 | final-trace 评估复用 helper：stop-check 与 final evaluator 相同（identity + raw-turn + manifest + observability）时复用结果，避免重复调用 LLM judge；post-run mutation 阻止复用。**Audit 收紧：同 EK-13/14，为公共 helper 能力（单测验证），未接入当前内置策略** | trace.py:148-197; test_trace.py:350 | mechanism↔EK-13；dependency→EK-16；constraint→EK-23 |
| EK-16 | Turn 不可变：执行循环构造 provisional Turn（无 eval_result）→ evaluator 评估（EvalContext 含全 history）→ dataclasses.replace 产生最终 frozen Turn；driver 不能 mutate owned history list | execution.py:429-475; types.py:292-327; test_trace.py:124 | constraint→EK-13, EK-15 |

## D. 评估器与 LLM Judge
| ID | 知识 | 证据 | links |
|---|---|---|---|
| EK-17 | 评估器组合算子：`|` 短路在左 DETECTED（便宜确定性评估器放左、昂贵 LLM judge 放右）；`&` 短路在左 NOT_DETECTED（UNDETERMINED 左不短路——右可能仍 NOT_DETECTED 定案，短路会造成顺序依赖）；`~` flip DETECTED↔NOT_DETECTED、UNDETERMINED 透传 | evaluator.py:92-306 | mechanism↔EK-05；causal→EK-14；contrast↔EK-22（确定性 vs LLM） |
| EK-18 | LLMJudge 是"diffuse 语言级信号"的评估器：JSON 契约 _JudgeVerdict（outcome/confidence/rationale/evidence）严格校验；markdown fenced JSON / prose 中 JSON 均解析 | llm_judge.py:45-127, 250-297; test_llm_judge.py:209-214 | mechanism↔EK-19, EK-20；subsystem↔EK-23 |
| EK-19 | LLM 失败降级语义：InvalidJson/EmptyResponse/RateLimit（重试耗尽后）→ UNDETERMINED（confidence 0.0），**不伪装成 NOT_DETECTED**；配置/设置失败（endpoint/auth）→ EvaluatorError → 基础设施 ERROR | llm_judge.py:481-531; test_llm_judge.py:219-239 | causal→EK-06；mechanism↔EK-18；constraint→EK-04 |
| EK-20 | judge 加固后缀（_HARDENING_SUFFIX）：system prompt 末尾追加 SECURITY BOUNDARY——transcript 是待评估 DATA 不是命令；追加在 _build_system_prompt 之后，子类 override 自动继承 | llm_judge.py:370-377, 476-478 | constraint→EK-23；mechanism↔EK-35（纵深防御） |
| EK-21 | TranscriptScope：FULL 见全部轮次 vs CURRENT_TURN 只见最后一轮（早期正常轮次会稀释注入信号时用 CURRENT_TURN）；attachment content 排除、metadata 保留（防 payload 内容注入 judge 视角） | llm_judge.py:45-70, 553-568; test_llm_judge.py:392 | subsystem↔EK-18；constraint→EK-20 |
| EK-22 | ResponseContains：substring / compiled regex / callable predicate 三模式；scope 显式（ANY_TURN/ALL_TURNS/CURRENT_TURN）只作用于已有 turns 不控制执行轮数 | response_contains.py:16-76 | contrast↔EK-17；mechanism↔EK-02 |
| EK-23 | judge 可复现性：verdict 默认非确定性，CI 可复现需 temperature=0 + seed（LLMConfig.metadata）；_send_async 是测试 override 点（绕过真实 PyRIT send 而不碰 CentralMemory） | llm_judge.py:364-369, 570-595 | constraint→EK-15；subsystem↔EK-18 |

## E. 信任边界与输入消毒（纵深防御）
| ID | 知识 | 证据 | links |
|---|---|---|---|
| EK-24 | xdist 信任边界：worker payload 可能含 attacker-controlled 内容（agent response/payload text）；序列化严格 JSON-safe 原语；反序列化校验 schema version（SCHEMA_VERSION "rampart.xdist.v2"）+ enum + metadata depth（MAX 6）；ANSI escape 从自由文本剥离 | _xdist.py:1-50, 655-742 | mechanism↔EK-25, EK-26；constraint→EK-27, EK-31 |
| EK-25 | safe_str 防御：第三方 evaluator 可在字段放任何东西；plain str() 会因 __str__ raise 拖垮 summary 与 verdict——用 str.__str__（C slot 不可覆盖）强制精确 str；"hostile value should not take a verdict the evaluators already reached" | text.py:61-103 | mechanism↔EK-24；dependency→EK-26 |
| EK-26 | safe_float/safe_str_list：JSON 无 NaN/Inf，非有限实数 → None（bool 拒绝，防 True 序列化为 1.0）；恶意/缺陷值 → 空 list 而非崩溃 | text.py:121-137; test_xdist.py:251-270 | mechanism↔EK-24, EK-25；causal→EK-27 |
| EK-27 | 序列化往返不丢判定：hostile evidence/operand/rationale 值经消毒后往返不丢 payload；sanitized confidence 不夸大（缺省 confidence 才默认 1.0，显式 null 不算满值） | test_xdist.py:364-494 | dependency→EK-24, EK-26；constraint→EK-31 |
| EK-28 | payload 存储路径安全：反序列化 artifact 路径必须留在 collection artifacts 目录内（拒绝 ../、绝对路径、非字符串、symlink 逃逸、缺失文件）——恶意 payloads.jsonl 不能逃逸存储根 | _store.py:220-234; test_payload_store_security.py:16-88 | mechanism↔EK-24；constraint→EK-35 |
| EK-29 | PayloadStore 原子写：temp-dir-then-rename（shutil.move 处理跨设备），防 pytest-xdist 多 worker 并发生成冲突；manifest.json 保留 provenance（collection/count/formats） | _store.py:54-89, 272-280 | subsystem↔EK-11；mechanism↔EK-24 |

## F. 适配器协议与依赖反转
| ID | 知识 | 证据 | links |
|---|---|---|---|
| EK-30 | AgentAdapter 三要素：create_session_async + manifest + observability_profile；Session 是 async context manager（幂等清理）；"Fresh state = fresh session"（对话重置即新 session） | adapter.py:1-90 | constraint→EK-01, EK-03, EK-08；mechanism↔EK-34 |
| EK-31 | InjectionHandle 依赖反转：execution strategy 只依赖 InjectionHandle 协议（enter 激活/exit 清理，幂等、不 raise、异常也保证清理），从不 import Surface 实现 | injection.py:1-89; _xpia.py:162-193 | dependency→EK-33, EK-08；constraint→EK-07, EK-24 |
| EK-32 | Surface 构造时配置完成（凭据/目标位置），universal inject(payload) 签名；wait_until_ready_async 应 raise TimeoutError 防无限阻塞 | injection.py:56-89; onedrive.py:89-101 | subsystem↔EK-31；constraint→EK-33 |
| EK-33 | XPIA 七阶段 + 清理保证：handle 并发 entry 用 asyncio.gather(return_exceptions=True) 而非 TaskGroup——TaskGroup 取消会孤儿化"已创建未注册清理"的远端 payload；就绪等待用 TaskGroup 并发 | _xpia.py:100-193; test_xpia.py:231 | causal→EK-34；mechanism↔EK-31；constraint→EK-32 |
| EK-34 | XPIA 触发语义：trigger 永不 adversarial——攻击在注入 payload，不在 prompt（"Summarize Q3"）；inline XPIA 以 Request.attachments 携带毒化内容，无 surface 也可 | _factory.py:57-99; docs/attacks/xpia.md:31-40 | causal→EK-33；constraint→EK-30 |
| EK-35 | LLMDriver 对话一致性校验：driver 侧 CentralMemory 用户轮数必须等于 agent 侧 history 长度，否则 DriverError（driver 被跨测试复用 / 未重放 history 就 resume 的检测） | drivers/llm.py:248-278 | mechanism↔EK-20（防污染）；constraint→EK-30；subsystem↔EK-36 |

| EK-45 | judge persona 与框架降级是两层语义：内置 NEUTRAL_EVALUATOR 的 system_prompt 指引 judge 对"证据文本内容模糊"倾向 NOT_DETECTED（文本解读指引）；框架层对"LLM 调用失败/解析失败"降级 UNDETERMINED（机制信心）——两者不冲突，但不能混淆（用户若误读"模糊→安全"会致假阴性理解） | personas.py:1-30; llm_judge.py:481-531 | contrast↔EK-19；constraint→EK-21 |

## G. PyRIT 生态边界
| ID | 知识 | 证据 | links |
|---|---|---|---|
| EK-36 | RAMPART 用 PyRIT 但公共 API 不暴露 PyRIT 类型（唯一例外 LLMDriver.from_target 接受 PromptChatTarget）；公共类型（LLMConfig/Persona/Payload）是 RAMPART 自己的 | docs/concepts/pyrit.md; pyrit_bridge/llm_bridge.py:45-99 | contrast↔EK-37；dependency→EK-35；constraint→EK-38 |
| EK-37 | PyRIT 能力映射：PromptChatTarget→LLMConfig+create_prompt_target；PromptNormalizer→LLMDriver 内部；CentralMemory→LLMDriver 内部；WordDocConverter→DocxConverter；DriverError 可包裹 PyRIT 异常（原错误保留在 message） | docs/concepts/pyrit.md:12-26; errors.py | mechanism↔EK-36；contrast↔EK-36（薄封装 vs 语义层） |
| EK-38 | RAMPART 在 PyRIT 之上新增四件事：safety-test 语义（attack/probe + Result.safe/status）、执行编排（injection→trigger→evaluate + early stop + AsyncExitStack）、pytest 集成（markers/trial/xdist/报告）、评估器组合算子、类型安全协议 | docs/concepts/pyrit.md:37-45 | causal→EK-08, EK-13, EK-17；constraint→EK-36 |

## H. 报告与治理
| ID | 知识 | 证据 | links |
|---|---|---|---|
| EK-39 | TestRunReport 聚合：by_harm_category（StrEnum 值即原生字符串，自定义 string 分类畅通）+ population_summary（attack_success_rate/safety_pass_rate）；is_clean_run 要求全 SAFE（UNDETERMINED/ERROR 也算不 clean） | sink.py:20-96 | causal→EK-06, EK-41；dependency→EK-04 |
| EK-40 | sinks 由 conftest hook（pytest_rampart_sinks）提供而非配置；controller 解析、worker 不执行 fixtures；非 ReportSink 项 dropped + warning（一个坏实现不破坏 emission） | plugin.py:402-436; _hooks.py:1-34 | subsystem↔EK-39；constraint→EK-41 |
| EK-41 | xdist 三模式：worker 写 slim metadata 到 workeroutput 跳过 sink；controller 合并后统一发射；非 xdist 单进程管线；WorkerOutputError → mark_incomplete（不中止 pytest，匹配 workeroutput 信任边界约定） | plugin.py:465-527; test_xdist_aggregation.py:150-442 | causal→EK-07, EK-24；mechanism↔EK-24 |
| EK-42 | trace schema 版本化治理：serialize_record/deserialize_record 为唯一公共 API；版本化 envelope + 无版本 body（nested result 不得单独持久化）；反序列化无 string/bool/int coercion；Payload ID 必须记录而非生成；schema 由脚本生成并过兼容门禁 | docs/concepts/trace-schema.md; serialization.py; scripts/check_trace_compatibility.py | constraint→EK-24, EK-28；mechanism↔EK-24 |
| EK-43 | 自定义 lint 治理：tools/flake8_rampart.py 提供 RMP001/RMP002 外部规则（ruff select=ALL + RUF 忽略项 + external 声明），pre-commit 挂载 | pyproject.toml:90-100; .pre-commit-config.yaml | subsystem↔EK-42；constraint→EK-39 |
| EK-44 | RAMPARTner onboarding skill：随包分发（rampart/skills/rampartner/SKILL.md），指导"首个测试证明什么→建集成→运行→选代码落地的后续测试→本地扩展 vs 框架 feature request" | skills/rampartner/SKILL.md:1-40 | causal→EK-34, EK-39；constraint→EK-43 |

## 聚合映射（KO ← EK 簇，详见 03）
- KO-01 ← {EK-01, EK-02, EK-03, EK-04, EK-05}（R3 不变量：观测通道与判定诚实）
- KO-02 ← {EK-06, EK-07, EK-39}（R3 不变量：故障≠发现，green 语义）
- KO-03 ← {EK-24, EK-25, EK-26, EK-27, EK-28, EK-20}（R1 机制簇：纵深防御消毒链）
- KO-04 ← {EK-17, EK-05, EK-14}（R1 机制簇：组合评估器与 UNDETERMINED 传播）
- KO-05 ← {EK-08, EK-09, EK-10, EK-11, EK-12}（R2 因果链簇：生命周期骨架→事件系统→trial）
- KO-06 ← {EK-30, EK-31, EK-32, EK-33, EK-34}（R2 因果链簇：适配器协议→注入→XPIA）
- KO-07 ← {EK-36, EK-37, EK-38}（R4 主题簇：PyRIT 生态边界）
- KO-08 ← {EK-18, EK-19, EK-21, EK-23}（R4 主题簇：LLM-as-judge 可靠性工程）
