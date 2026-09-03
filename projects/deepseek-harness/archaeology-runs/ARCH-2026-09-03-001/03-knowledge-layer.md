# Knowledge Layer — deepseek-harness（Generalized KO）

> Job: `ARCH-2026-09-03-001` | 层：Generalized Knowledge（窄尖顶，L3/L4）| 8 个 KO
> 每个 KO 必须声明 `aggregation_rule`（R1-R4），从 EK 图簇生成；derivation.facts 恰等于簇内 EK。
> 升维经 Abstraction Promotion Gate 独立判定（见 06）。未跨项目验证的原则标 Cross-project validation pending。

---

## KO-01 — 日志驱动的可恢复 Agent 回合循环（R2 因果链簇）

- **claim**：Agent 回合循环以**不可变会话日志**为中心——turn/step 每个边界都写日志，模型历史从日志派生、结果从日志判定、持久化从日志重放；因此回合循环天然**可重放、可恢复、可审计**。
- **knowledge_layer**: generalized | **category**: RUNTIME/MEMORY | **abstraction**: L4 | **value**: A
- **epistemic_status**: Validated Pattern | **confidence**: high
- **aggregation_rule**:
  - rule: **R2**（因果链簇：决策→执行→记录）
  - cluster_eks: [EK-01, EK-02, EK-03, EK-06]
  - edges: causal EK-01→EK-02→EK-03；EK-06 约束 EK-03 的格式演进
  - naming: "回合循环 = 日志边界"——所有 EK 共享"先写日志再判定"的因果链
  - scope_expansion: 任一成员 EK 只描述单点（循环/日志/持久化/版本），KO-01 断言三者组合构成"可恢复循环"这一整体性质
- **derivation.facts**: [EK-01, EK-02, EK-03, EK-06]
- **flow_traceability**: F-02（State）turn/step 状态迁移；F-03（Data）日志→消息派生
- **scope**: applies_when Agent runtime 需要恢复/审计/重放；does_not_apply_when 无状态单次调用
- **validation**: truth PASS | flow VERIFIED | counterexample none(见 06 C-05 反例) | blind_reconstruction performed

---

## KO-02 — Capability Seam：可插拔能力的统一三件套（R1 机制簇）

- **claim**：把每个可插拔能力（LLM、沙箱、凭证、子代理…）建模为 **Service Definition / Provider / Consumer** 三件套，并让依赖关系**由代码生成图**（而非手维护），是规模化 Agent 框架保持可替换性的稳定架构模式。
- **knowledge_layer**: generalized | **category**: ARCHITECTURE | **abstraction**: L4 | **value**: A
- **epistemic_status**: Validated Pattern | **confidence**: high
- **aggregation_rule**:
  - rule: **R1**（机制簇：同一机制跨 ≥2 独立子系统）
  - cluster_eks: [EK-23, EK-16, EK-12, EK-13, EK-21, EK-22]
  - edges: mechanism EK-23↔EK-16↔EK-12↔EK-13↔EK-21↔EK-22（同为 seam 三件套实例；EK-22 为 subagent seam 的外部 provider 家族）
  - naming: "三件套"模式在 llm/sandbox/credentials/subagent（含外部 agent provider）多个独立子系统重复出现
  - scope_expansion: EK-23 描述机制本身，KO-02 断言"多系统各自独立演化仍保持统一可替换契约"的系统级性质
- **derivation.facts**: [EK-23, EK-16, EK-12, EK-13, EK-21, EK-22]
- **flow_traceability**: F-05（Authority）沙箱/凭证/子代理均为 provider 层
- **scope**: applies_when 需要为多个外部/内部实现提供可替换点；does_not_apply_when 单一实现无替换需求
- **validation**: truth PASS | abstraction OK | counterexample none(见 C-03) | blind_reconstruction performed

---

## KO-03 — 权限系统的 Fail-Closed 不变量族（R3 不变量簇）

- **claim**：Agent 权限体系把"默认拒绝"实现为多个**独立层叠的 fail-closed 不变量**——审批非 allowed-once 即拒绝、工具守卫只有拒绝没有允许、沙箱三态非允许即受限、凭证空值即不存在——**任意一层失败都不产生授权**。
- **knowledge_layer**: generalized | **category**: PERMISSION | **abstraction**: L4 | **value**: A
- **epistemic_status**: Principle（项目内验证，Cross-project validation pending）| **confidence**: high
- **aggregation_rule**:
  - rule: **R3**（不变量簇：多 EK 汇聚到同一安全不变量）
  - cluster_eks: [EK-10, EK-09, EK-07, EK-13]
  - edges: constraint/mechanism EK-09→EK-07；EK-10↔EK-09（fail-closed 家族）；EK-13 独立层
  - naming: 四层都指向同一不变量"默认拒绝，失败=拒绝"
  - scope_expansion: 任一 EK 只覆盖单层，KO-03 断言"层叠安全"（defense in depth）系统性质
- **derivation.facts**: [EK-10, EK-09, EK-07, EK-13]
- **flow_traceability**: F-05（Authority）审批→守卫→沙箱→凭证四段
- **scope**: applies_when Agent 执行不可信输入、持敏感能力；does_not_apply_when 无权限语义的只读工具
- **validation**: truth PASS | abstraction OK | counterexample found→已条件化（见 C-06 danger-full-access）| blind_reconstruction performed

---

## KO-04 — 四层上下文记忆治理（R4 主题簇）

- **claim**：Agent 的长上下文记忆由四层各司其职治理——**日志层**（不可变事实）、**surface 层**（模型可见投影，可阴影替换）、**投影层**（派生状态）、**压缩层**（summary 节点替换历史）——每一层都从日志派生且保持可回滚。
- **knowledge_layer**: generalized | **category**: MEMORY/CONTEXT | **abstraction**: L4 | **value**: A
- **epistemic_status**: Validated Pattern | **confidence**: high
- **aggregation_rule**:
  - rule: **R4**（主题簇：同一主题互补维度）
  - cluster_eks: [EK-02, EK-04, EK-05, EK-18, EK-19, EK-20]
  - edges: EK-02→EK-04→EK-19（causal/constraint）；EK-05 与 EK-19 共享投影/事件机制；EK-18/EK-20 为上下文组装与回合结果约束
  - naming: 六条 EK 覆盖"事实→可见→派生→压缩"记忆生命周期的互补维度
  - scope_expansion: 任一 EK 只描述一层机制，KO-04 断言四层如何协同且互不越权
- **derivation.facts**: [EK-02, EK-04, EK-05, EK-18, EK-19, EK-20]
- **flow_traceability**: F-06（Memory）日志→surface→投影→压缩
- **scope**: applies_when 长会话、token 预算受限、需要审计；does_not_apply_when 无状态短会话
- **validation**: truth PASS | coverage PASS | counterexample none(见 C-05) | blind_reconstruction performed

---

## KO-05 — 日志即真相：可重放策略与审计（R3 不变量簇）

- **claim**：**"model-visible means logged"**——模型可见、策略决策、审计事件全部只能通过不可变日志进入/重建；没有任何旁路状态；因此会话的完整因果链（含安全决策）可被重放验证。
- **knowledge_layer**: generalized | **category**: MEMORY/EVIDENCE | **abstraction**: L4 | **value**: A
- **epistemic_status**: Principle（项目内验证，Cross-project validation pending）| **confidence**: high
- **aggregation_rule**:
  - rule: **R3**（不变量簇：跨层实现同一不变量）
  - cluster_eks: [EK-02, EK-04, EK-11]
  - edges: constraint EK-02→EK-04；subsystem EK-11（approval/policy 事件）同属日志
  - naming: 三 EK 都实现"日志是唯一真相"——模型可见（surface）、策略（approval/policy）、审计（asked/decided）
  - scope_expansion: 从"日志存数据"上升到"日志是系统认知的唯一权威来源"
- **derivation.facts**: [EK-02, EK-04, EK-11]
- **flow_traceability**: F-04（Evidence）日志→重放→验证；F-07（Policy）策略重放
- **scope**: applies_when 需要审计/合规/调试重放；does_not_apply_when 性能敏感无需追溯
- **validation**: truth PASS | counterexample none(见 C-05) | blind_reconstruction performed

---

## KO-06 — 策略治理闭环：一次性决策 → 持久策略 → 未来决策（R2 因果链簇）

- **claim**：系统把一次性安全决策固化为**持久策略**（approval policy / sandbox mode / permission preset 写入日志），并让该策略**约束未来的决策**（never 政策拒绝后续审批、preset 决定后续沙箱行为）——形成完整的政策治理闭环。
- **knowledge_layer**: generalized | **category**: POLICY/GOVERNANCE | **abstraction**: L4 | **value**: A
- **epistemic_status**: Validated Pattern | **confidence**: high
- **aggregation_rule**:
  - rule: **R2**（因果链簇：Decision→Approval→Policy→Enforcement→Future）
  - cluster_eks: [EK-10, EK-11, EK-14, EK-12]
  - edges: causal EK-10→EK-11→EK-14→EK-12（决策→策略→预设→执行边界）
  - naming: 与 Policy Flow（F-07）同构的标准治理链
  - scope_expansion: 从单点审批上升到"策略如何自我固化并约束未来"的闭环
- **derivation.facts**: [EK-10, EK-11, EK-14, EK-12]
- **flow_traceability**: F-07（Policy）
- **scope**: applies_when Agent 需要按会话/场景改变权限规则；does_not_apply_when 一次性执行无持久策略
- **validation**: truth PASS | flow VERIFIED | counterexample none | blind_reconstruction performed

---

## KO-07 — 结构化失败族：让错误可路由、可归属、可重放（R1 机制簇）

- **claim**：Agent 运行时把失败建模为**结构化错误族**（专用 code：TOOL_TIMEOUT / SANDBOX_UNAVAILABLE / UNKNOWN…），每个错误可归属到确切故障层、可被重试/回放/沙箱插件路由；broad 签名规则会导致错误归属失真（landlock 教训）。
- **knowledge_layer**: generalized | **category**: FAILURE | **abstraction**: L3 | **value**: A
- **epistemic_status**: Validated Pattern | **confidence**: high
- **aggregation_rule**:
  - rule: **R1**（机制簇：同一机制跨多子系统）
  - cluster_eks: [EK-17, EK-08, EK-15]
  - edges: mechanism EK-17↔EK-08↔EK-15（结构化错误 + 归属纪律）
  - naming: 三 EK 共享"错误必须结构化且归属可信"机制
  - scope_expansion: 从单点错误处理上升到"错误归属影响诊断/重放正确性"系统性质
- **derivation.facts**: [EK-17, EK-08, EK-15]
- **flow_traceability**: F-04（Evidence）错误 code→重放路由
- **scope**: applies_when 多层抽象叠加（runtime/sandbox/LLM）；does_not_apply_when 单层简单工具
- **validation**: truth PASS | counterexample found→条件化（SEARCH_FAILED 曾掩盖错误，见 EK-15）| blind_reconstruction performed

---

## KO-08 — 可插拔系统的治理护栏：插件树 + 生成图 + 运行时不变量 + 启动强制（R4 主题簇）

- **claim**：一个以可插拔为架构主轴的系统，需要配套**四类治理护栏**才能防止"插件自由"退化成混乱：插件内核（everything-is-a-plugin）、自动生成的依赖图（gen-doc-graphs）、包自拥有的运行时不变量（invariants）、应用启动边界强制（verify-application-entrypoints）。
- **knowledge_layer**: generalized | **category**: ARCHITECTURE/GOVERNANCE | **abstraction**: L4 | **value**: A
- **epistemic_status**: Validated Pattern | **confidence**: high
- **aggregation_rule**:
  - rule: **R4**（主题簇：同一主题互补维度）
  - cluster_eks: [EK-24, EK-23, EK-25, EK-26]
  - edges: EK-24→EK-23（dependency）；EK-25→EK-26（constraint）；EK-26→EK-24（constraint）
  - naming: 四 EK 覆盖"组装方式→图治理→运行时自检→启动边界"四个互补维度
  - scope_expansion: 从各自护栏上升到"护栏组合如何让可插拔系统保持可治理"整体性质
- **derivation.facts**: [EK-24, EK-23, EK-25, EK-26]
- **flow_traceability**: F-01（Control）启动装配
- **scope**: applies_when 大型可插拔框架（插件生态）；does_not_apply_when 单体小库
- **validation**: truth PASS | abstraction OK | counterexample none | blind_reconstruction performed

---

## 层间分布
| 层 | 数量 | 对象 |
|---|---|---|
| L3 Pattern | 1 | KO-07 |
| L4 Cognitive Model | 7 | KO-01..06, KO-08 |
| L5 Methodology | 0 | （无跨项目证据，全部标 pending，不升 L5） |
| Cross-project validation pending | 2 | KO-03, KO-05 |
