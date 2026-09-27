# 02 · Engineering Knowledge — EK Graph（AProver）

> 21 条 EK，六类边（mechanism/subsystem/causal/dependency/constraint/contrast），每条必须声明 links。
> 证据来源一律标注文件/行/README 节；未满足者降 D 级不入图。反退化铁律：无 links 即模块说明。

---

## EK-01：删除不对称性（健全性第一定律）
- **内容**：从报告中排除一个发现是"报告集合收窄"，收窄是非对称的——错误排除会静默隐藏真实 bug（假阴性 = 健全性损失），错误保留只增加噪声（精度损失）。健全性只要求一个具体性质：**任何真实 bug 永不被静默删除**。
- **证据**：`soundness_policy.py` L1-31（模块 docstring 原话 "no real bug is ever silently removed from the report"；"narrowing is asymmetric"）
- **links**：mechanism(EK-02, EK-03)；causal→EK-04（删除权限决定五档分级）；constraint→EK-07（UNRESOLVED 不静默丢弃）

## EK-02：三种判定类型 → 权限映射
- **内容**：删除/降级的判定按证据性质分三类：`DETERMINISTIC_VERIFIER`（CBMC 重验证明排除）→ 可 DELETE；`SELF_VERIFYING_WITNESS`（编译+精确复现的确定性事实）→ 可 DELETE；`AGENTIC_JUDGMENT`（LLM 判定，非确定性、可能自信地错）→ 只能 RE-TIER（降级），**永不可作为唯一删除依据**。
- **证据**：`soundness_policy.py` L37-67（Justification/Action 枚举 + resolve_action；"an LLM whose mistakes can only cost precision"）
- **links**：mechanism(EK-01)；causal→EK-04（档位降级路径）；constraint→EK-05（refiner 删除需双条件）

## EK-03：refiner 排除的授权条件（CBMC 排除 ∧ 调用点确定性检查）
- **内容**：spec-refiner 通过**添加**前置条件子句排除反例时：CBMC 重验只证明该子句**排除** CEx，不证明子句在**每个调用点**成立——后者是非确定性的 SoundnessAgent 判断。故仅当 BOTH（CBMC 排除成立）AND（子句在所有调用者处被确定性检查）才允许 DELETE，否则 RE-TIER。
- **证据**：`soundness_policy.py` L70-86（refiner_exclusion_action）；`cex_validator.py` _refine_precondition / _agentic_soundness_guard（L1692/L1818）
- **links**：constraint→EK-02；causal→EK-08（精化循环的门控）；mechanism(EK-05)

## EK-04：五档证据分级 + realism 降级免疫
- **内容**：confirmed_dynamic（GCC harness 运行时故障）＞ confirmed_system_entry（调用链回溯到无调用者函数）＞ confirmed_bmc（至少一个直接调用者可达 CEx 状态）＞ likely（过度精化守卫触发，假定真实以防压制）＞ unlikely（realism 判定 UNREALISTIC）。**confirmed_dynamic 免疫 realism 降级**，唯一例外是 `main.assertion.N` 属性（LLM 生成的 postcondition 编码 = harness 噪声，非源码语义）。
- **证据**：README L40-48；PIPELINE.md Tier assignment；`bug_reporter.py` L156-171（tier 赋值链）；realism 免疫段 PIPELINE.md L151-153
- **links**：causal←EK-02/EK-03；subsystem(EK-10, EK-19)；constraint→EK-06（动态验证是最高档的来源）

## EK-05：反例三态——UNRESOLVED 永不被静默丢弃
- **内容**：CEx 验证三态：REAL_BUG（具体化成功且违规复现）/ SPURIOUS（具体化成功但不复现）/ **UNRESOLVED（具体化失败——绝不静默丢弃，跟踪跳过）**。SPURIOUS 时 Spec Refiner（agentic）提议更紧前置条件。
- **证据**：`cex_validator.py` L1-9（模块 docstring "UNRESOLVED — concretization fails after all attempts (never silently dropped)"）；PIPELINE.md Phase 3 分支
- **links**：mechanism(EK-01)；causal→EK-08；subsystem(EK-06)

## EK-06：动态验证 = 自验证证人（确定性事实）
- **内容**：Phase 3 S3 用 GCC 编译运行复现 harness，signal handler 捕获 SIGSEGV/SIGABRT/SIGFPE/SIGILL；判定 CONFIRMED（harness 触发信号 = 运行时确认）。生成可以非确定，**编译+运行是确定性事实**——故 witness 可 DELETE（精确复现同一故障、经真实公共 API、匹配 CBMC 属性）。
- **证据**：`dynamic_validator.py` L1-10（模块 docstring）、L244-268（signal_name / EVIDENCE-QUALITY 信号区分 harness-artifact vs real-bug-signal）；`soundness_policy.py` L89-96（witness_action）
- **links**：causal→EK-04（confirmed_dynamic 来源）；constraint→EK-02；subsystem(EK-05)

## EK-07：每函数隔离验证 + __CPROVER_assume callee stub（compositional）
- **内容**：每个函数隔离验证：callee 被替换为受 LLM 生成的 postcondition 约束的 stub（`__CPROVER_assume`）；BMC 后端用该 spec + stubs 检查函数。跨文件调用者经两遍全局调用图构建追踪，避免函数被误判为系统入口。
- **证据**：README L36（"Each function is verified in isolation…"）；harness_generator.py / dsl_to_cbmc.py（__CPROVER_assume 翻译）
- **links**：mechanism(EK-12)；dependency→EK-09（stub postcondition 的质量约束 spec 生成）；subsystem(EK-08)

## EK-08：CEGAR 精化循环 + Soundness Guard（over-refinement 检查）
- **内容**：SPURIOUS → Refiner 提议更紧前置条件 → Soundness Guard：BMC over-refinement 检查（LLM fallback 兜底）→ accepted（精化 spec）→ Phase 3c 重跑该函数 + Phase 3b 重跑其调用者（compositional propagation）→ 回到 Phase 3（capped re-queue）。最多 `BMC_AGENT_MAX_REFINEMENT_ITERS=5` 轮。
- **证据**：README L30-33（Phase 3b/3c）；PIPELINE.md（Soundness Guard 框 + 3c/3b CEGAR）；`cex_validator.py` _check_over_refinement_with_cbmc（L1876）/ _check_over_refinement（L1991）；config.py BMC_AGENT_MAX_REFINEMENT_ITERS
- **links**：causal→EK-07（stub 精化后调用者重验）；mechanism(EK-03)；constraint←EK-01

## EK-09：SCC 分层拓扑生成顺序
- **内容**：spec 生成按调用图分层拓扑序：先 Kosaraju 求 SCC → 压缩为 SCC-DAG → BFS/Kahn 分层 → 展平。Layer 1 = 无调用者的入口函数，逐层下推——保证 callee 的 spec 先于 caller 生成（组合式 spec 生成的前提）。
- **证据**：`spec_generator.py` _kosaraju_sccs（L145）/ _build_generation_order（L226-260，docstring 完整算法）
- **links**：causal→EK-07（先 callee 后 caller 的 spec 依赖）；mechanism(EK-12)

## EK-10：dual-spec 双重生成 + 分歧标记
- **内容**：`BMC_AGENT_ENABLE_DUAL_SPEC=true`（默认）时，每个 spec 以两种不同强调生成两次（caller-heavy / impl-heavy），检测分歧则标记 `spec_disagreement`；dual 失败回退单次生成。设计意图：捕获 caller 契约与实现不一致的典型假阳性类（"textbook caller-contract-slip FP class"）。
- **证据**：`spec_generator.py` _generate_dual_spec（L1343-1449，含分歧检查 L1420-1440）；L6（模块 docstring "caller-heavy and impl-heavy sources to flag disagreements"）；L89-112（_permissive_spec 注释）
- **links**：mechanism(EK-13)；constraint→EK-01（分歧标记 = 不静默压制）；subsystem(EK-16)

## EK-11：spec-DSL 设计——LLM 可发射、无解释器介入、直接翻译
- **内容**：规格用小型 DSL 表达（valid/valid_string/valid_range/in_bounds/null/owns/locked/\result），正则直接翻译为 CBMC 构造（如 valid_string→ptr!=NULL+harness 有界缓冲；valid_range→ptr!=NULL && lo>=0 && hi>=lo）；不匹配模式的自然语言子句作为 `/* comments */` 发射（`BMC_AGENT_STRICT_DSL` 强制推入 JSON reasoning）。设计目标：LLM 可靠发射 + 零解释器干预。
- **证据**：README L231-245（Specification DSL 表）；`dsl_to_cbmc.py` L18-31（谓词正则 + 翻译注释）、L219-289（translate_atom 实现）
- **links**：dependency→EK-07（stub 翻译）；constraint→EK-10（DSL 可靠性是 dual-spec 前提）；subsystem(EK-14)

## EK-12：两遍全局调用图 + 系统入口判定
- **内容**：跨文件调用者经两遍全局调用图构建追踪；"confirmed_system_entry" 档位 = 全语料中无任何调用者的函数（完整调用链回溯）。`_is_publicly_callable`（非 static 函数）参与可达性判定。防止：函数被误判为系统入口 / 虚假系统入口。
- **证据**：README L36、L42-43（confirmed_system_entry 定义）；`cex_validator.py` _is_publicly_callable（L105-108）；pipeline.py cross_file_callers
- **links**：mechanism(EK-09)；causal→EK-04（system_entry 档位来源）；subsystem(EK-07)

## EK-13：realism witness-pattern 确定性短路
- **内容**：realism 检查在调 LLM 前做确定性预检，命中模式直接返回 UNREALISTIC（省 LLM 往返 + 避免 LLM"创造性地"为人为产物辩护）：
  - **未初始化库全局**：CBMC nondet 默认值表明 bug 需要库未初始化（真实 init 后不可能）
  - **jq jv stub-disconnect**：CBMC nondet `jv` struct u.ptr=NULL 而 stub `jv_get_kind` 返回 refcnt-backed kind——stub 断开的产物
  - **NULL-guard violation**：函数体有 `if(!ptr) return 0;` 显式守卫但反例显示守卫指针为 NULL 且后续行报解引用违规——CBMC 符号执行未按守卫路径条件剪枝（路径发散）
  - 另有 USB-serial 框架不变量（pl2303 来源）
- **证据**：`realism_checker.py` L150-235（三个预检块 + 各自来源注释 2026-05-13/05-18）
- **links**：mechanism(EK-14, EK-02)；contrast→EK-15（短路 vs 完整 LLM 审计）；causal→EK-04（UNREALISTIC→unlikely 降档）

## EK-14：确定性检查优先于 LLM 判定的通用模式
- **内容**：项目在多处把"能确定性判定的"提前到"LLM 判定"之前：realism 短路（EK-13）、Soundness Guard 用 BMC 而非 LLM（EK-08）、属性类免疫（EK-17）、CEx dedup（EK-18）。共同机制：**用最确定的可用工具在流水线最早点消除不确定性**。
- **证据**：跨模块横切观察（soundness_policy / realism_checker / pipeline / cex_validator 结构对比）
- **links**：mechanism(EK-13, EK-02, EK-17)；contrast→EK-15

## EK-15：realism 完整审计（LLM 层）与降档保留
- **内容**：未被短路命中的 REAL_BUG 走 LLM 审计：verdict ∈ {realistic / unrealistic / uncertain}；UNREALISTIC 降级为 unlikely（**不是丢弃**），保留完整审计轨迹；UNCERTAIN 保留但注解。budget 耗尽/llm 出错时 pass-through 为 REALISTIC（solver verdict 权威）。
- **证据**：`realism_checker.py` L52-148（RealismVerdict/RealismCheckResult/check 入口 + pass-through 行为）；README L48（"downgraded to unlikely rather than discarded, preserving the full audit trail"）
- **links**：contrast→EK-13/EK-14；causal→EK-04；constraint←EK-01

## EK-16：测试-实现-文档三方漂移（agentic realism 默认值）
- **内容**：`--agentic` 模式下：测试 `test_agentic_keeps_classifier_on_realism_tools_on_triage_off_dynval_on` 期望 `enable_realism_check=True`，而实现返回 False。README §Agentic mode 明确"realism + triage are OFF by default and independently opt-in"——**实现符合 README，测试未同步更新**。本机 pytest 实测确认（213 passed / 1 failed）。
- **证据**：tests/test_agentic_components.py L29 断言失败；README L143-154（agentic 模式段落）；pytest 实测输出
- **links**：contrast→EK-21（README 驱动的开发流程假设）；subsystem(EK-10)

## EK-17：属性类免疫（property-class exclusion）
- **内容**：同一 (function, property_class) 下，若某 CEx 经 soundness-gated 精化前置条件驱动到 VERIFIED CLEAN，本轮 sweep 剩余同属性类兄弟 CEx 被排除（"remaining same-class siblings are excluded by it"）。属性类键 = SVCOMP_PROP token（真实代码→"memsafety"；`_prop_is_reach` 只在 SVCOMP_PROP=unreach 时为真——属性驱动非 benchmark 门控）。
- **证据**：`pipeline.py` L1032-1045（跳过逻辑 + 日志文案）；L40-47（_prop_is_reach + SVCOMP_PROP 注释）
- **links**：mechanism(EK-14, EK-08)；constraint→EK-01（免疫基于 soundness-gated 精化）

## EK-18：CEx 去重（每 (function, property_type) 一个）
- **内容**：`_dedup_counterexamples`：同一 (function, property_type) 只保留一个反例，`assertion.N` 保持完整保留；默认 max_per_type（DEFAULT_DEDUP_PER_TYPE）。
- **证据**：`pipeline.py` L303-342（_dedup_counterexamples）；PIPELINE.md（CEx Dedup 框）
- **links**：mechanism(EK-14)；constraint→EK-05（去重不丢 UNRESOLVED）

## EK-19：spec_evidence 调用者证据收集（v2 spec gen）
- **内容**：v2 spec 生成前 harvest 调用者证据：CallerEvidence（调用点）+ address_taken_sites + DocClause/SeedClause/FieldAccessHint + 测试路径过滤（_is_test_path）+ 字符串字面量剥离 + 注释行过滤——给 LLM 的"调用者视角"证据束，供 spec 生成时考虑真实调用约束。
- **证据**：`spec_evidence.py` L36-140（EvidenceBundle 结构）、L209-317（harvest_callers）
- **links**：causal→EK-09（spec 生成输入）；subsystem(EK-10)

## EK-20：ML 内核验证里程碑（scale-down / safety-only 参数化）
- **内容**：对 ML 内核（llm.c train_gpt2.c）使用参数化缩小验证：`--infer-field-validity`（struct primitive-pointer 字段初始化为 NULL-or-malloc，使 `if(field!=NULL)` 守卫不被 nondet-invalid 状态击败）、`--infer-array-param-bounds`（指针参数 backing 数组按函数体最大字面量下标定长，cap _MAX=64）、`--scale-down`（参数化尺寸 B/T/C/NH 界到 [0,_SIZE] 默认 4 + 自动开数组边界；止 matmul/attention 超时）、`--safety-only`（postcondition 限内存安全+范围+NaN/Inf 自由，无功能声明）。效果：22/30 clean（up from 4/30），B=T=C=NH=4 缩放尺寸。
- **证据**：README L215-219（配置表）、L251（llm.c 段落）；findings/llm_c/scorecard_m1_m12_m2.json
- **links**：mechanism(EK-14)；constraint→EK-04（safety-only 影响可验证属性集）；subsystem(EK-16)

## EK-21：README 驱动的开发流程（文档即规范）
- **内容**：README/PIPELINE.md 描述的实现与代码一致（本机验证 soundness_policy 行为、agentic 默认值、管线阶段均符合文档）；文档以"as implemented"自述（PIPELINE.md 标题）。开发流程明显为文档驱动（docs 提交常见：HEAD commit 即 docs: add findings）。
- **证据**：PIPELINE.md 标题（"as implemented"）；HEAD commit 主题（docs）；README 配置表与 config.py 一致
- **links**：contrast→EK-16（文档-测试漂移的对照面）；subsystem(EK-10)

---

## EK Graph 结构总结
- **subsystem 簇**：验证引擎（EK-04/05/06/07/08/12）、规格引擎（EK-09/10/11/19）、判定治理（EK-01/02/03/13/15）、流水线优化（EK-14/17/18）
- **causal 主链**：EK-01 → EK-02/03 → EK-08 → EK-07 → EK-04（健全性约束 → 判定授权 → 精化门控 → 组合验证 → 分级输出）
- **contrast 对**：EK-13↔EK-15（确定性短路 vs 完整审计）、EK-16↔EK-21（测试漂移 vs 文档驱动）
- **游离 EK**：无（全部 ≥1 边）
