# 03 · Knowledge Layer — Generalized KO（AProver）

> 7 个 KO，每个声明 aggregation_rule（R1-R4）与簇内 EK；升维经 Abstraction Promotion Gate 独立判定。
> 铁律：L3+ 必须解释范围扩大；单一项目证据不写成普遍定律（标 Cross-project validation pending）。

---

## KO-01：LLM 判定只能降级，不能删除（R3 不变量簇）
- **epistemic_status**: Pattern（L3）｜ **aggregation_rule**: **R3 不变量簇**
- **簇内 EK + 边**：EK-01（删除不对称性，constraint 边）→ EK-02（判定类型→权限映射，mechanism 边）→ EK-03（refiner 双条件，constraint 边）→ EK-05（UNRESOLVED 不丢弃，mechanism 边）→ EK-15（realism 降档保留，causal 边）
- **陈述**：在验证/审计系统中，"从结论集删除一项"与"降低其置信"是不同权限等级的操作；任何非确定性判定（LLM）只能降级，删除必须由确定性验证或确定性证人支撑——否则系统会静默隐藏真实缺陷。
- **解释范围扩大论证**：不限于本项目的 bug 报告——适用于任何 agent 决策系统（安全拦截、内容审核、代码审查）中"置信阈值以下的内容如何处置"问题；Aigis 的 CaMeL 无条件 DENY（09-27 考古）与本项目 soundness_policy 是同一原则在"执行阻断"与"证据分级"两个面的实例。
- **反例预算（≥3）**：① JIT 上下文场景可能"宁可删除"（假阴性代价低于噪音）——需显式成本函数而非一律保留；② 若下游有二次人工复核，LLM 删除可被兜底；③ 纯性能优化场景（非安全）删除成本不对称性不成立。→ 均不推翻"安全报告语境下"的陈述，但说明适用范围需限定"不可逆决策输出"。

## KO-02：agentic model checking 的信任链（R2 因果链簇）
- **epistemic_status**: Cognitive Model（L4）｜ **aggregation_rule**: **R2 因果链簇**
- **簇内 EK + 边**：EK-09（SCC 分层生成）→ EK-10（dual-spec）→ EK-11（DSL 直接翻译）→ EK-07（隔离验证 + stub）→ EK-08（CEGAR + Soundness Guard）→ EK-04（五档分级）
- **陈述**：在 LLM+形式工具混合验证中，信任沿"规格生成（agentic）→ 翻译（确定性）→ 求解（形式）→ 分类（混合）→ 分级（确定性优先）"逐层传递；每一层的不确定性被下一层更确定的检查兜底，最终输出档位由最接近确定性事实的环节决定。
- **解释范围扩大论证**：这是"验证编译器"（Claim→Operationalization→Evidence→Judgment）的机器实现形态——任何"AI 提主张、机器给证据"的系统（代码审查、合规检查、医学辅助诊断）共享此信任链结构。
- **反例预算**：① 信任链假设 solver 本身可信——CBMC 自身 bug 不在模型内（Kani/CBMC 均为独立项目，未做元验证）；② Frama-C/WP 反向合成路径的"数学整数 vs 机器整数"二义由 overflow-rigor 缓解但未完全消除（README 承认 math-int only 情况）；③ per-role provider 混用（anthropic+claude-code+codex）引入跨提供者一致性风险。

## KO-03：确定性短路——用模式库替代 LLM 判定（R1 机制簇）
- **epistemic_status**: Pattern（L3）｜ **aggregation_rule**: **R1 机制簇**
- **簇内 EK + 边**：EK-13（realism witness-pattern 短路，mechanism 边）↔ EK-14（确定性优先通用模式，mechanism 边）↔ EK-17（属性类免疫，mechanism 边）↔ EK-18（CEx 去重，mechanism 边）
- **陈述**：同一机制跨 ≥2 独立子系统出现：在 LLM 判定之前，用确定性模式/检查在流水线最早点消除可消除的不确定性——命中已知模式直接给结论（省 LLM 往返 + 避免 LLM 创造性合理化人为产物）。
- **解释范围扩大论证**：任何高成本 LLM 调用前做"模式库预检"的系统（日志分类、威胁检测、代码分类）均可复用；模式库随实测失败演进（realism_checker 注释记录 05-13/05-18 事件驱动的模式添加）。
- **反例预算**：① 模式库膨胀 → 维护成本与误命中（模式过宽会误判真实 bug 为 artifact——realism 的 UNREALISTIC 短路以"high confidence"标记，仍可被下游覆盖）；② 新模式发现依赖真实失败驱动，冷启动阶段覆盖不足；③ 跨项目迁移性存疑（jq 特有 stub-disconnect 不适用于非 jq 代码）→ 见 C1。

## KO-04：每函数隔离 + compositional 传播（R4 主题簇）
- **epistemic_status**: Pattern（L3）｜ **aggregation_rule**: **R4 主题簇**
- **簇内 EK + 边**：EK-07（隔离验证 + __CPROVER_assume stub，subsystem 边）→ EK-09（SCC 分层，causal 边）→ EK-12（调用图 + 系统入口，mechanism 边）→ EK-08（3c 自验 + 3b 调用者传播，causal 边）→ EK-19（调用者证据，subsystem 边）
- **陈述**：大规模验证通过"函数级隔离 + 契约化 stub + 精化后传播"控制组合爆炸：callee 用其 postcondition 作为 stub 契约，caller 独立验证；规格精化后只重验受影响函数及其调用者，而非全量重跑。
- **解释范围扩大论证**：模块化验证/测试/类型系统的通用策略（单元测试+契约、增量重编译、类型传播）；"改一个契约影响所有调用者"是软件工程的通用问题。
- **反例预算**：① stub postcondition 质量差 → 假阳性传播（README 承认 spec 质量是可选 Phase 4，未默认开启）；② SCC 环内函数（递归）的组合验证语义弱于 acyclic 图；③ 跨进程/动态加载边界（dlopen）不在调用图内。

## KO-05：证据分级 = 信任分层（L4 Cognitive Model）
- **epistemic_status**: Cognitive Model（L4）｜ **aggregation_rule**: **R3 不变量簇**
- **簇内 EK + 边**：EK-04（五档分级，constraint 边）→ EK-06（动态验证确定性事实，causal 边）→ EK-12（系统入口回溯，causal 边）→ EK-15（realism 降档，constraint 边）
- **陈述**：系统对结论的信任强度应与其证据的"确定性距离"单调对应：从运行时复现（确定性事实）→ 调用链回溯（结构事实）→ BMC 可达性（半形式）→ 守卫推断（代理）→ LLM 审计（非确定），档位逐级下降；越级提升需额外证据。
- **解释范围扩大论证**：与 KO-01 互补——前者管"如何删"，本模型管"如何信"；适用任何证据驱动的判定系统（司法证据、医学诊断、CI 门禁）。Aigis 的审计决策解耦（09-27）同构。
- **反例预算**：① "确定性距离近 ≠ 相关性强"——动态复现的崩溃可能来自 harness 噪声（项目用 main.assertion.N 例外处理）；② 用户可能偏好 recall 优先（宁多报）——档位设计隐含 precision 倾向。

## KO-06：智能 ≠ 权威（验证语境）（L4 Cognitive Model）
- **epistemic_status**: Cognitive Model（L4）｜ **aggregation_rule**: **R3 不变量簇**
- **簇内 EK + 边**：EK-01（删除不对称性，constraint 边）→ EK-02（权限映射，mechanism 边）→ EK-15（降档保留，causal 边）
- **陈述**：判断力（智能，能提出规格/分类反例）与处置权（权威，能删除结论）是两个独立维度：agent 可以最有智能，但不因此获得删除权威；删除权威只授予确定性证据。
- **解释范围扩大论证**：与 Tafcm/Codex 考古的 "Intelligence ≠ Authority"（skill 内置示范）及 Aigis 的 CaMeL 无条件 DENY 同构——**这是跨三个独立项目的收敛**：执行阻断（Aigis）、规格验证（AProver）、受控工具（Tafcm）。Cross-project validation: 3 项目收敛（S8 级证据），但作为 Principle 仍需更多领域（金融/医疗）实例。
- **反例预算**：① 某些场景智能与权威必须合一（外科手术 agent、单点决策）——受限于"证据可逆性"假设；② 若 LLM 判定带确定性后验（如形式化输出的 LLM 证明检查器），可重分类为 deterministic——边界由实现定义。

## KO-07：混合验证分工方法论（L5 Methodology）
- **epistemic_status**: Methodology（L5）｜ **aggregation_rule**: **R2 因果链簇**
- **簇内 EK + 边**：EK-11（DSL 直接翻译，dependency 边）→ EK-14（确定性优先，mechanism 边）→ EK-16（文档-实现-测试对齐，contrast 边）→ EK-20（参数化缩小验证，mechanism 边）→ EK-21（文档即规范，contrast 边）
- **陈述**：构建 LLM+传统工具混合系统时：（1）语义任务给 LLM、形式保证给 solver、删除决策只给确定性证据；（2）LLM 接口用"可靠可发射的小 DSL"而非自由文本（翻译零解释器介入）；（3）能确定性判定的尽早短路；（4）大规模/参数化场景用"缩小+安全子集"先建立基线再扩张（llm.c 22/30 路径）；（5）文档（README/PIPELINE as-implemented）是规范的权威载体，测试须与文档-实现三方一致（本机已发现一处漂移 EK-16）。
- **解释范围扩大论证**：可操作准则（能做）：为任何"AI+确定性工具"系统设计时按此五条分工；本方法论由 AProver 单一项目导出 → **Cross-project validation pending**（标 Hypothetical 边界）。
- **反例预算**：① 缩小验证（scale-down）不证明全尺寸正确（overflow-rigor 部分缓解）；② 文档驱动开发在高速迭代期会产生文档滞后（EK-16 即证据）；③ DSL 约束可能遗漏 LLM 能表达但 DSL 不能捕获的规格（strict-dsl 把 prose 推入 reasoning 字段即承认）。

---

## KO 聚合规则矩阵
| KO | 规则 | 簇内 EK 数 | 解释范围扩大 | 反例数 |
|---|---|---|---|---|
| KO-01 | R3 | 5 | 通用审计/决策系统 | 3 |
| KO-02 | R2 | 6 | 验证编译器/证据系统 | 3 |
| KO-03 | R1 | 4 | 高成本 LLM 预检系统 | 3 |
| KO-04 | R4 | 5 | 模块化验证/测试/类型 | 3 |
| KO-05 | R3 | 4 | 证据驱动判定系统 | 2 |
| KO-06 | R3 | 3 | 权限设计（3 项目收敛） | 2 |
| KO-07 | R2 | 5 | 混合系统设计方法论 | 3 |

无"同子系统=聚合理由"的假聚合；全部 KO 可回溯 EK/证据（见 06 追溯表）。
