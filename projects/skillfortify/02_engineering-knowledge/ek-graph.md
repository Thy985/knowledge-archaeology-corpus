# 02 Engineering Knowledge — EK Graph（宽底座）

> 36 条 EK，每条带 `links`（6 类边：mechanism/subsystem/causal/dependency/constraint/contrast）。证据来源均可回溯 dbb5942。认知状态见每条 epistemic（Fact=实现已读/实测；Obs=文档声称）。Cross-project 候选见 05。

## EK-01 保守过度近似能力推断（sound abstract interpretation）
- **内容**：StaticAnalyzer._infer_capabilities 用保守过度近似推断能力："if a pattern suggests a capability, we include it. False positives are acceptable; false negatives are not."
- **证据**：src/skillfortify/core/analyzer/engine.py L150-193（docstring 原文）
- **实测**：仅含 URL 的 skill → {network:READ}；含 `curl -X POST` → {network:WRITE, shell:WRITE}
- **epistemic**: Fact
- **links**: [mechanism: EK-02], [causal: →EK-04], [constraint: 约束 EK-05 的"无假阴性"声明]

## EK-02 能力格 lattice（join/meet/bottom/top）
- **内容**：AccessLevel{NONE<READ<WRITE<ADMIN} 构成格，join=max、meet=min、bottom=NONE、top=ADMIN；类方法注释声明交换/结合/幂等/单位元性质。
- **证据**：core/capabilities/levels.py L4-144；CapabilitySet per-resource 取最高（join）
- **实测**：join(READ,WRITE)=WRITE, meet(WRITE,ADMIN)=WRITE
- **epistemic**: Fact
- **links**: [mechanism: EK-01], [subsystem: EK-02/EK-03 同 capabilities 子系统]

## EK-03 不可变 Capability + CapabilitySet POLA 核对
- **内容**：Capability 是 (resource,access) 不可变对（frozen=True），注释明确实现 Dennis & Van Horn (1966) capability 概念并"防止 TOCTOU"；CapabilitySet 提供 permits/is_subset_of/violations_against（形式化 POLA）。
- **证据**：core/capabilities/models.py L49-116
- **epistemic**: Fact
- **links**: [subsystem: EK-02], [dependency: EK-03 依赖 EK-02 的格语义]

## EK-04 三阶段分析管线（推断→危险模式→违反核对）
- **内容**：analyze() 三段：Phase1 capability inference → Phase2 dangerous pattern detection → Phase3 capability violation check（inferred vs declared）；dangerous-pattern 不作为 soundness 依据（注释明确：over-declaration/unparsable 在 Phase3 查）。
- **证据**：analyzer/engine.py L107-137
- **实测**：声明 filesystem:read 但含 `rm -rf /` → safe=False，2 条 privilege_escalation（CRITICAL+HIGH）
- **epistemic**: Fact
- **links**: [causal: EK-01→EK-04→EK-06], [contrast: EK-05]

## EK-05 危险模式检测是"召回型"非"soundness 依据"
- **内容**：patterns.py 含 _DANGEROUS_SHELL_PATTERNS/_DANGEROUS_CODE_PATTERNS/_PROMPT_INJECTION_PATTERNS/_SENSITIVE_ENV_PATTERNS 等模式目录；engine 注释把 dangerous-pattern 结果与 capability-soundness 明确分开（Pattern 检测可以有假阴性，capability 推断不允许）。
- **证据**：patterns.py L179-479；engine.py L127-137 注释
- **epistemic**: Fact
- **links**: [contrast: EK-04], [mechanism: EK-07]

## EK-06 声明vs实际能力违反 → privilege_escalation
- **内容**：当 inferred 超 declared 时产出 privilege_escalation finding（实测 CRITICAL/HIGH 双条）。
- **证据**：实测 + analyzer/models.py Finding.attack_class
- **epistemic**: Fact（本项目实测）
- **links**: [causal: EK-04→EK-06], [constraint: EK-03 的 violations_against]

## EK-07 反规避归一化（零宽字符/同形字/Unicode）
- **内容**：patterns.py 定义 _ZERO_WIDTH_CHARS、_HOMOGLYPH_MAP、normalize_for_matching，匹配前 Unicode 归一化；用 `_EVAL_NAME="ev"+"al"` 拼接规避"自我模式被 grep 误报"。
- **证据**：patterns.py L106-136；测试 test_zero_width_space_obfuscation_is_detected
- **epistemic**: Fact
- **links**: [mechanism: EK-05], [causal: 反规避→EK-05 覆盖]

## EK-08 双用途分级（inline interpreter：ordinary tooling 非 critical vs 携带 payload critical）
- **内容**：测试断言普通工具链上的内联解释器不判 CRITICAL，携带恶意载荷则保持 CRITICAL——分析器区分"工具存在"与"工具被用于攻击"。
- **证据**：tests/core/analyzer/test_dual_use_grading.py（test_inline_interpreter_on_ordinary_tooling_is_not_critical / test_inline_interpreter_carrying_a_payload_stays_critical）
- **epistemic**: Fact
- **links**: [contrast: EK-05（模式检测 vs 载荷判断）], [mechanism: EK-07]

## EK-09 DY-Skill：Dolev-Yao 攻击者适配 skill 供应链
- **内容**：DYSkillAttacker 五能力 intercept/inject/synthesize/decompose/replay，消息=SkillMessage，网络=SupplyChain(author→registry→developer→environment)；维护单调知识集 K（Monotonicity/Interception closure/Synthesis closure 三不变量）；synthesize/replay 拒绝未知消息（DY closure violation）。
- **证据**：threat_model/dy_skill.py 全文；threat_model/messages.py
- **epistemic**: Fact
- **links**: [causal: EK-09→EK-10], [mechanism: EK-02（都是形式模型）]

## EK-10 攻击分类学（AttackClass×ThreatActor×AttackSurface×AttackType A1-A13）
- **内容**：taxonomy.py 定义攻击类（data_exfil/priv_esc/prompt_injection/dep_confusion/typosquatting/namespace_squatting）、威胁角色、攻击面、A1-A13 攻击类型；测试断言"mapping exact per paper 32"与 registry-dependent 分类（typosquatting/dep_confusion/namespace_squatting 需 registry 可观测；data_exfil/priv_esc/prompt_injection 不需）。
- **证据**：threat_model/taxonomy.py；tests/core/threat_model/test_taxonomy.py
- **epistemic**: Fact
- **links**: [causal: EK-09→EK-10], [constraint: registry-dependent 类约束 EK-25 的 registry 扫描必要性]

## EK-11 信任代数：intrinsic 加权 + 依赖 min 传播 + 时间衰减 + 证据更新
- **内容**：T_intrinsic = Σ w_i·T_i（四信号，clamp [0,1]）；T_effective = T_intrinsic × min(dep.effective)（最弱链）；apply_decay 时间衰减（测试：69 天减半、230 天到 10%）；update_with_evidence 正证据单调递增、负证据拒绝、clamp 到 1。
- **证据**：trust/engine.py L102-175、trust/propagation.py；tests/core/trust/*.py
- **实测**：intrinsic=0.90 + dep effective=0.2 → effective=0.18
- **epistemic**: Fact
- **links**: [mechanism: EK-12], [causal: EK-11→EK-13]

## EK-12 信任单调性（Theorem 5 的实现侧）
- **内容**：update_with_evidence 保证正证据不减分；测试含 hypothesis 性质测试 test_monotonicity_property_all_signals。
- **证据**：trust/propagation.py L128-146；tests/core/trust（monotonicity 系列）
- **epistemic**: Fact
- **links**: [mechanism: EK-11], [constraint: EK-12 约束 EK-11 的可预测性]

## EK-13 score_to_level（TrustScore→L0-L3）
- **内容**：compute_score 末尾 score_to_level(effective) 映射信任级别；wiki Trust-Levels.md 定义阈值。
- **证据**：trust/engine.py L181-233；wiki/Trust-Levels.md
- **epistemic**: Fact（级别映射实现），Obs（阈值文档）
- **links**: [causal: EK-11→EK-13]

## EK-14 SHA-256 完整性锁文件（SRI 风格）+ SOURCE_DATE_EPOCH 可重现
- **内容**：Lockfile.compute_integrity 对完整 skill 内容做 SHA-256（Subresource Integrity 风格）；_generation_timestamp 由 SOURCE_DATE_EPOCH 锚定，保证提交到 VCS 的锁文件可字节级重现（CHANGELOG 0.6.0：regenerate byte-identically）。
- **证据**：lockfile/lockfile.py L1-52,133-137；CHANGELOG 0.6.0
- **epistemic**: Fact
- **links**: [mechanism: EK-15（都是供应链固定机制）], [causal: EK-14→EK-15]

## EK-15 Agent SBOM（ASBOMGenerator）
- **内容**：core/sbom 用 cyclonedx-python-lib 生成 Agent 软件物料清单（add_component/add_from_parsed_skill/generate/to_json/write_json）。
- **证据**：sbom/generator.py；docs/asbom.md；wiki/ASBOM-Guide.md
- **epistemic**: Fact
- **links**: [mechanism: EK-14], [subsystem: EK-14/EK-15 同 lockfile/sbom 输出面]

## EK-16 统一有界树遍历 discovery（单次扫描覆盖所有格式）
- **内容**：CHANGELOG 0.6.0：所有 skill 格式共享一个 bounded tree walk；OpenClaw install roots/MCP 配置在 scan root 下任意深度找到；MCP server 按项目目录去重（同名的不同包均报告）。
- **证据**：CHANGELOG 0.6.0；discovery/system_scanner.py（_find_mcp_configs/_find_skill_dirs）
- **epistemic**: Fact
- **links**: [causal: EK-16→EK-04（发现→分析）], [constraint: bounded walk 约束 EK-25 的扫描范围]

## EK-17 系统级发现（SystemScanner 探测 IDE 配置）
- **内容**：SystemScanner 扫描 home 下已知/未知 IDE 的 skill/MCP 配置（_discover_known_ides/_discover_unknown_ides/_probe_profile）。
- **证据**：discovery/system_scanner.py L41-232
- **epistemic**: Fact
- **links**: [subsystem: EK-16], [causal: EK-17→EK-16]

## EK-18 probe→parse 两阶段解析器注册表
- **内容**：SkillParser ABC 定义 can_parse（快速探测，仅查文件/目录存在）+ parse（完整解析）；ParserRegistry 高效试多个 parser。
- **证据**：parsers/base.py L84-110（两阶段 API 注释）
- **epistemic**: Fact
- **links**: [causal: EK-18→EK-01（解析→分析）]

## EK-19 威胁模式的"精确文本化"防御（字符串拼接规避 grep 匹配）
- **内容**：`_EVAL_NAME = "ev" + "al"`、`_EXEC_NAME = "ex" + "ec"` —— 模式库自身避免被文本搜索误判为真实恶意代码（也防攻击者简单 grep 即得规避线索）。
- **证据**：patterns.py L364-365
- **epistemic**: Fact
- **links**: [mechanism: EK-07], [contrast: EK-08]

## EK-20 声明工具授权（allowed-tools/disallowed-tools）作为能力读取
- **内容**：CHANGELOG 0.5.0：declared tool grants 被读作 capabilities 并与推断行为核对（扩展声明vs实际核对到工具粒度）。
- **证据**：CHANGELOG 0.5.0
- **epistemic**: Obs（CHANGELOG 声称，未逐行重读实现）
- **links**: [causal: EK-20→EK-06（都是声明核对）]

## EK-21 registry 抓取加固（重定向重验证/公开地址检查/响应大小上限）
- **内容**：CHANGELOG 0.5.0：registry fetches 增加 redirect revalidation、public-address checks、response size caps。
- **证据**：CHANGELOG 0.5.0
- **epistemic**: Obs（CHANGELOG 声称）
- **links**: [constraint: 加固约束 EK-25 的 registry 扫描], [dependency: EK-21 依赖 httpx（可选依赖 registry）]

## EK-22 树级完整性哈希（覆盖整个 skill 目录而非仅 SKILL.md）
- **内容**：CHANGELOG 0.5.0：tree-wide integrity hashing over a skill directory rather than SKILL.md alone。
- **证据**：CHANGELOG 0.5.0
- **epistemic**: Obs（CHANGELOG 声称）
- **links**: [mechanism: EK-14]

## EK-23 反泄漏基准设计（结构特征不能预测标签）
- **内容**：SkillFortifyBench 540-skill（270 恶意 13 类型 + 270 良性 5 类），seed=42 确定性生成；corpus_leakage 测试：`test_no_structural_feature_predicts_the_label` / `test_both_classes_draw_names_from_one_vocabulary` / `test_specimens_share_one_installation_layout`（防"标签泄漏进结构"的机械检查）。
- **证据**：benchmarks/generator/config.py；tests/core/benchmark_generator/test_corpus_leakage.py；README "mechanical check"
- **epistemic**: Fact
- **links**: [mechanism: EK-24], [causal: EK-23→EK-24]

## EK-24 基准生成防写越界（_assert_safe_output_root）
- **内容**：BenchmarkGenerator._assert_safe_output_root 校验输出根在项目根内，防基准生成器把恶意/良性样例写到任意路径。
- **证据**：benchmarks/generator/core.py L134
- **epistemic**: Fact
- **links**: [constraint: EK-24 约束 EK-23 的落盘安全]

## EK-25 形式化模型已知局限（显式边界）
- **内容**：Formal-Foundations §Known Limitations 四条：①install-time 攻击（typosquatting/dep confusion）需 registry 级分析，超出本地静态分析范围（对应 6% recall gap）；②runtime 行为（静态分析推理"能做什么"非"将做什么"）；③语义理解（资源级能力，不推理数据语义敏感度）；④混淆（重度混淆抗静态分析）。
- **证据**：wiki/Formal-Foundations.md §Known Limitations
- **epistemic**: Obs（作者自述局限）
- **links**: [constraint: EK-25 约束 EK-04 的结论范围], [contrast: EK-21（registry 扫描部分缓解①）]

## EK-26 文档声称"无过度近似" vs 实现"保守过度近似"（设计-实现差距）
- **内容**：Formal-Foundations Theorem 2 章节声称 "The analysis can compute the exact capability requirements of a skill **without over-approximation or under-approximation** at the capability level"；实现（EK-01）明确是 conservative over-approximation。over-approximation 恰是 soundness 来源，但文档措辞错误。
- **证据**：wiki/Formal-Foundations.md（Theorem 2 段落）vs analyzer/engine.py L153-158
- **epistemic**: Fact（两处原文并列，矛盾客观存在）
- **links**: [contrast: EK-01], [causal: EK-26→Candidate C-01]

## EK-27 GitNexus 工程治理（AGENTS.md 影响分析门）
- **内容**：AGENTS.md 强制编辑前 gitnexus_impact（upstream blast radius）并报告；HIGH/CRITICAL 风险必须警告；提交前 gitnexus_detect_changes；索引：9441 symbols/17297 relationships/225 execution flows。
- **证据**：AGENTS.md（gitnexus:start 块）
- **epistemic**: Fact
- **links**: [constraint: EK-27 约束项目内所有代码编辑], [contrast: EK-28]

## EK-28 SECURITY.md 漏洞披露治理
- **内容**：支持 0.1.x；in-scope 列表（错误安全判定/能力推断绕过/锁文件完整性/供应链/恶意当安全）；out-of-scope（第三方依赖/超大输入 DoS known limitation/物理访问）；48h 确认/5 工作日评估承诺；coordinated disclosure。
- **证据**：SECURITY.md
- **epistemic**: Fact
- **links**: [contrast: EK-27], [constraint: EK-28 约束漏洞报告的接收路径]

## EK-29 Elastic-2.0 许可证（候选卡记 MIT 的 discrepancy）
- **内容**：pyproject/license=Elastic-2.0（classifiers 亦标注 Other/Proprietary License）；候选卡 [cand]skill-supply-chain-security-2026 记 MIT，与实际不符。
- **证据**：pyproject.toml license 字段 + classifiers；LICENSE 文件
- **epistemic**: Fact
- **links**: [contrast: EK-30（合规面）]

## EK-30 版本语义化 + 实验可重现性治理
- **内容**：CHANGELOG 按 SemVer；0.6.0 加 `python -m benchmarks.experiments` 写 benchmarks/results/experiments.json（含 host），`make experiments` 跑全套——实验测量可重现、可归因主机。
- **证据**：CHANGELOG 0.6.0；benchmarks/experiments/host.py
- **epistemic**: Fact
- **links**: [mechanism: EK-14（可重现理念同源）]

## EK-31 四信号信任语义与权重可配置
- **内容**：TrustWeights 可配置（engine.__init__），compute_intrinsic 用权重；四信号 Provenance/Behavioral/Community/Historical 定义见 Formal-Foundations Trust Algebra 表。
- **证据**：trust/engine.py L79-96；wiki/Formal-Foundations.md Trust Algebra
- **epistemic**: Fact（权重实现），Obs（信号语义文档）
- **links**: [mechanism: EK-11]

## EK-32 依赖解析（图/约束/解析器 + SAT 可选）
- **内容**：core/dependency 含 graph/constraints/resolver；pyproject 可选 sat 依赖（python-sat）；Theorem 4（Resolution Soundness）声明 SAT 求解同时满足版本/冲突/安全约束；Formal-Foundations 明确"npm/pip 启发式回溯不能处理安全约束"。
- **证据**：core/dependency/*；pyproject.toml optional-dependencies；wiki/Formal-Foundations.md §SAT
- **epistemic**: Fact（模块存在），Obs（SAT 声明未实测——python-sat 可选未装）
- **links**: [causal: EK-32→EK-14（解析→锁文件）]

## EK-33 错误处理与 CLI 健壮性（test_error_handling）
- **内容**：tests/cli/test_error_handling.py 存在，覆盖 CLI 错误路径。
- **证据**：tests/cli/test_error_handling.py
- **epistemic**: Fact（存在性），未逐行验证覆盖
- **links**: [subsystem: cli 面]

## EK-34 dashboard HTML 报告生成
- **内容**：cli/dashboard_cmd + dashboard/generator（HTML 安全报告）；README quick start `skillfortify dashboard`。
- **证据**：cli/dashboard_cmd.py；dashboard/generator.py；README
- **epistemic**: Fact
- **links**: [causal: EK-04→EK-34（分析结果→报告）]

## EK-35 能力推断的四个启发式（URL/Shell/Env/File 模式）
- **内容**：_infer_capabilities 仅 4 类启发式：URLs→network（默认 READ，POST-like 升 WRITE）、shell→shell:WRITE、env ref→environment:READ、file write/read 模式→filesystem（write 优先于 read）。这是 soundness 的"近似层"——4 类之外的资源类型不被推断。
- **证据**：analyzer/engine.py L162-191
- **epistemic**: Fact
- **links**: [constraint: EK-35 是 EK-04 soundness 的实际边界（只覆盖 4 资源）], [causal: EK-35→Candidate C-03]

## EK-36 benchmark 生成器确定性（seed=42 + generation_order）
- **内容**：BenchmarkGenerator._generation_order 迭代（确定性生成顺序）；config 指定攻击类型与良性类；README 声明 "Deterministic from seed=42"。
- **证据**：benchmarks/generator/core.py L394；benchmarks/generator/config.py；README
- **epistemic**: Fact
- **links**: [mechanism: EK-23], [causal: EK-36→EK-23]

---

## EK 边统计（防退化检查）

- 总 EK：36
- 含 links 的 EK：36/36（100%）
- 游离 EK（无出边）：0
- 平均出边：~2.1
- 聚合规则覆盖率：见 03（每个 KO 声明 R1-R4）
