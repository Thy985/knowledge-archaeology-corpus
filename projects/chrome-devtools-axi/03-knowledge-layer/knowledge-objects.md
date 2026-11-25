# 03 · Knowledge Layer — Generalized Knowledge Objects

> 窄尖顶。8 个 KO 全部从 EK 图上的簇生成；每个 KO 声明 `aggregation_rule`，`derivation.facts` 正好等于簇内 EK，不多不少。`abstraction` 标注 L3/L4；没有任何 KO 标 L5，也没有任何项目内结论被写成 Law。

## KO-01 · 跨进程连续性三件套（Continuity Triad）

```yaml
knowledge_layer: generalized
category: ARCHITECTURE
abstraction: L4
value: A
epistemic_status: Validated Pattern
provenance:
  discovered_by: engineering-knowledge-miner
  supported_by: [code-analyst, test-analyst, failure-analyst]
  contested_by: [abstraction-auditor, counterexample-hunter]
derivation:
  facts: [EK-03, EK-05, EK-11, EK-13, EK-15, EK-16, EK-17]
  promotion_arguments:
    - "L2→L3: 三个独立机制（deep health、selected page、snapshot generation）在不同文件、不同失效模式下重复出现，形成可命名模式。"
    - "L3→L4: 统一关系是'连续性不是一份状态，而是三份需要各自失效边界的交接'；它解释了为什么只检查 bridge 存活仍会 stale-ref / silent retarget。"
aggregation_rule:
  rule: R4
  cluster_eks: [EK-03, EK-05, EK-11, EK-13, EK-15, EK-16, EK-17]
  edges:
    - "subsystem: EK-03↔EK-05 / EK-03↔EK-11"
    - "causal: EK-15→EK-16 / EK-15→EK-17"
    - "dependency: EK-11→EK-13"
  naming: "三份连续性（endpoint / identity / evidence）共享同一个跨进程状态文件底座，但失效事件不同，故可命名。"
  scope_expansion: "任一成员只解释一份状态；KO 解释三者为何必须同时存在，并预测任一缺失会产生哪类静默错误。"
evidence:
  supporting: [EV-004, EV-005, EV-006, EV-027, EV-031, EV-033, EV-034, EV-035]
  contradicting: []
scope:
  applies_when: "短命 CLI / 多进程 wrapper 代理一个有状态浏览器或编辑器会话"
  does_not_apply_when: "单进程、单次请求、无跨命令路由状态的工具"
confidence: high
validation:
  truth: PASS
  coverage: PASS
  abstraction: OK
  counterexample: "3 次攻击：只检查 PID、只持久化 pageId 不做 epoch、只用时间戳判断新鲜度；三者都能在代码中找到反例，KO 已把失效边界写入 claim。"
  counterexample_budget: {required: 3, attempted: 3, searched_paths: [single-process, attach-mode, legacy-ref]}
  blind_reconstruction: performed
flow_traceability:
  l1_facts_to_edges:
    - {l1_fact: "deep health 调 list_pages", flow_edge: "F-C04: src/client.ts:473-506", verified: true}
    - {l1_fact: "selected page 由 select/new 写入", flow_edge: "F-S02: src/selected-page.ts:100-135", verified: true}
    - {l1_fact: "snapshot generation 每次抓取递增", flow_edge: "F-S03: src/generation.ts:20-40", verified: true}
```

**Claim**：一个跨越多次 CLI 调用的 wrapper，必须显式管理三份连续性：endpoint 存活、routing identity、evidence freshness。三者的失效事件不同，不能用一份“bridge 活着”的状态代替。缺 endpoint 检查会复用死 bridge；缺 identity epoch 会静默切 tab；缺 evidence epoch 会把旧 uid 当成新页面。

**Scope expansion**：EK-04/05 只解释“怎么复用 bridge”，EK-11/13 只解释“怎么处理页面”，EK-15/16/17 只解释“怎么判断 ref 新鲜”。KO 解释三者共享同一个跨进程文件底座，却各自需要独立失效边界；这能预测“健康检查全绿但仍然错”的组合故障。

**Counterexamples**：单进程工具不需要三份连续性；attach 到同一个外部浏览器的多个 session 无法靠 generation 防止 cross-talk（EK-10），所以 KO 的 scope 必须排除“多 session 共享一个浏览器”的场景。

## KO-02 · Origin gate 不是 caller auth（Loopback Confused Deputy）

```yaml
knowledge_layer: generalized
category: SECURITY
abstraction: L4
value: A
epistemic_status: Validated Pattern
provenance:
  discovered_by: authority-analyst
  supported_by: [code-analyst, failure-analyst, test-analyst]
  contested_by: [counterexample-hunter, epistemic-auditor]
derivation:
  facts: [EK-19, EK-20, EK-21, EK-22, EK-36]
  promotion_arguments:
    - "L2→L3: Host/Origin 拒绝、无 token、任意 roots、无 per-caller 审计四个机制在 bridge 的不同层重复暴露同一个边界缺口。"
    - "L3→L4: 稳定关系是'网络位置（loopback）与调用者身份是两个正交维度'；Host/Origin 只证明请求来自浏览器同源策略之外，不证明谁在调用。"
aggregation_rule:
  rule: R2
  cluster_eks: [EK-19, EK-20, EK-21, EK-22, EK-36]
  edges:
    - "causal: EK-19→EK-21→EK-22→EK-36"
    - "constraint: EK-19→EK-20（gate 依赖严格 Host 解析）"
  naming: "origin control / caller identity / roots authority / audit 构成一条完整因果链，可命名。"
  scope_expansion: "单个 EK 只描述一个控制；KO 解释它们组合后为什么仍是 confused deputy，并预测本地进程可越过的边界。"
evidence:
  supporting: [EV-015, EV-016, EV-017, EV-018, EV-019, EV-024, EV-036, EV-047]
  contradicting: []
scope:
  applies_when: "本地 loopback HTTP 控制面，且 /call 能执行高权工具（浏览器、文件、shell、数据库）；调用方具备 loopback 可达性且能设置 Host"
  does_not_apply_when: "控制面只暴露只读、低权、无副作用端点；或已有 mTLS/token/peer-credential 认证；或 OS/网络命名空间阻断了跨 caller 的 loopback 可达性"
confidence: high
validation:
  truth: PASS
  coverage: PASS
  abstraction: OK
  counterexample: "攻击 1：同机进程伪造 Host 直连 /call；攻击 2：请求体 roots=/ 放大文件写；攻击 3：浏览器扩展/本地代理设置 Host；前两者代码直接支持，第三项未运行验证。"
  counterexample_budget: {required: 3, attempted: 3, searched_paths: [same-host-process, arbitrary-roots, browser-extension, ipv6-parser]}
  blind_reconstruction: performed
flow_traceability:
  l1_facts_to_edges:
    - {l1_fact: "gate 在路由前返回 403", flow_edge: "F-A01: src/bridge.ts:531-546", verified: true}
    - {l1_fact: "/call 直接映射 MCP 工具", flow_edge: "F-A02: src/bridge.ts:470-520", verified: true}
    - {l1_fact: "roots 可来自请求体", flow_edge: "F-A04: src/bridge.ts:428-441", verified: true}
```

**Claim**：本地 loopback 控制面有两个正交的信任边界：**浏览器 origin**（Host/Origin 可以挡 DNS rebinding）和**本地 caller identity**（需要 token、peer credential、mTLS 或 socket 权限）。把前者当成完整边界，会让 bridge 成为 confused deputy：一个能建立 loopback 连接并可设置 Host 的本地调用方，能以 bridge 的工具权限行动。

**Scope expansion**：EK-19/20 只证明“浏览器侧被挡住”；EK-21/22 证明“本地侧没有身份，且 roots 由 caller 给”；EK-36 证明“没有 per-caller 审计”。KO 把四者连成一条因果链：origin gate 的严格性反而界定了它不保护什么，从而解释为什么 GHSA 修复是必要但不充分的。

**Counterexamples / 未验证**：本机进程攻击未在运行时执行；`Host` 伪造在 Node/curl 层可行，但没有在本轮启动 bridge 做 live 请求。浏览器扩展是否能设置 `Host` 未验证。多用户主机上其他 UID 是否能连 loopback 端口，取决于 OS/防火墙，未验证。KO 的 claim 已限定为“本地进程模型”，没有升级为“所有部署都必然可被攻击”。

## KO-03 · 只对自己托管的资源施加隔离（Custody-Scoped Policy）

```yaml
knowledge_layer: generalized
category: SECURITY
abstraction: L3
value: A
epistemic_status: Validated Pattern
provenance:
  discovered_by: authority-analyst
  supported_by: [code-analyst, test-analyst, doc-analyst]
  contested_by: [counterexample-hunter]
derivation:
  facts: [EK-10, EK-23, EK-24]
  promotion_arguments:
    - "L2→L3: session 隔离与 keychain 隔离在两个独立子系统重复同一机制：只对 AXI 启动的资源生效。"
    - "L3→L4: 稳定关系是'隔离强度必须匹配 custody（谁启动/谁拥有）'；对别人的浏览器施加 flag 是无效承诺，对自己的浏览器不施加则是真实风险。"
aggregation_rule:
  rule: R1
  cluster_eks: [EK-10, EK-23, EK-24]
  edges:
    - "mechanism: EK-10↔EK-23（只隔离自己启动的资源）"
    - "constraint: EK-23↔EK-24（transport 模式决定是否 launch）"
  naming: "两个独立子系统共享 custody 机制，故可命名。"
  scope_expansion: "单条 EK 只解释 session 或 keychain；KO 解释两者为何采用同一所有权规则，并预测 attach 模式不应假装隔离。"
evidence:
  supporting: [EV-020, EV-021, EV-045, EV-001]
  contradicting: []
scope:
  applies_when: "工具会启动或 attach 外部资源（浏览器、数据库、容器、VM）"
  does_not_apply_when: "资源始终由工具独占创建，或始终由外部 operator 全权管理"
confidence: high
validation:
  truth: PASS
  coverage: PASS
  abstraction: UNDER-JUSTIFIED
  counterexample: "攻击：在 attach 模式也追加 keychain flag 是否能'更安全'？代码/测试证明这会造成无效承诺，且浏览器归用户所有；反例支持 KO 而非推翻。但只有两个实例，L4 过强，降为 L3。"
  counterexample_budget: {required: 3, attempted: 3, searched_paths: [attach-keychain, session-attach, shared-mcp]}
  blind_reconstruction: performed
flow_traceability:
  l1_facts_to_edges:
    - {l1_fact: "launch 模式追加 keychain flag", flow_edge: "F-A05: src/bridge.ts:683-760", verified: true}
    - {l1_fact: "attach 模式不追加", flow_edge: "F-A05: src/bridge.ts:700-735", verified: true}
    - {l1_fact: "session 只隔离 bridge", flow_edge: "F-A06: src/sessions.ts:94-122", verified: true}
```

**Claim**：在本项目的两个隔离面（session 与 keychain）上，安全/隔离策略按 **custody** 施加：工具自己启动的资源，工具负责隔离；工具 attach 的资源，策略归资源 owner。把“隔离”做成无条件全局开关，会在 attach 模式产生无效承诺；反过来在 launch 模式省略隔离，则把真实风险留给用户。

**Counterexamples**：共享 MCP 服务的 operator 拥有 Chrome keychain 策略，AXI 不应替它加 flag；多 session attach 同一浏览器时，bridge 级隔离无法阻止浏览器状态 cross-talk。两个反例都被 KO 的 scope 吸收。

## KO-04 · 身份 epoch 边界必须 fail-closed，不得自动 retarget

```yaml
knowledge_layer: generalized
category: STATE
abstraction: L4
value: A
epistemic_status: Validated Pattern
provenance:
  discovered_by: failure-analyst
  supported_by: [code-analyst, test-analyst]
  contested_by: [counterexample-hunter]
derivation:
  facts: [EK-11, EK-12, EK-13, EK-14, EK-26]
  promotion_arguments:
    - "L2→L3: select/new 写入、list_pages 只展示、reconnect 丢弃、错误文本兜底、isError 失败五个机制形成一条完整链。"
    - "L3→L4: 稳定关系是'上游重发 id 时，旧 id 不再指向同一对象；proxy 不能把旧意图映射到新 id'。"
aggregation_rule:
  rule: R2
  cluster_eks: [EK-11, EK-12, EK-13, EK-14, EK-26]
  edges:
    - "causal: EK-13→EK-14→EK-26"
    - "constraint: EK-11→EK-13；EK-12→EK-11"
  naming: "selection→epoch detection→error 是完整因果链，可命名。"
  scope_expansion: "单条 EK 只解释一个防护；KO 解释 identity epoch 的完整失效链，并预测 silent retarget 的具体路径。"
evidence:
  supporting: [EV-027, EV-028, EV-029, EV-031, EV-032, EV-035]
  contradicting: []
scope:
  applies_when: "上游 id 在重连/重启后会被重发，且 id 空间是进程级"
  does_not_apply_when: "id 是稳定 UUID 且跨重启不变；或调用方每次显式传 target"
confidence: high
validation:
  truth: PASS
  coverage: PASS
  abstraction: OK
  counterexample: "攻击 1：从 list_pages 的 [selected] 重建 id；攻击 2：reconnect 后把旧 id 映射到同 URL tab；攻击 3：把 missing-page 文本当成功。三者均被代码/测试拒绝。"
  counterexample_budget: {required: 3, attempted: 3, searched_paths: [list-pages-rebuild, url-remap, missing-page-success]}
  blind_reconstruction: performed
flow_traceability:
  l1_facts_to_edges:
    - {l1_fact: "reconnect 后显式 pageId 返回 identity error", flow_edge: "F-S02: src/bridge.ts:470-520", verified: true}
    - {l1_fact: "missing page 只匹配最后一行", flow_edge: "F-E05: src/client.ts:790-850", verified: true}
```

**Claim**：当上游重连/重启会重发标识符时，proxy 必须把它当成 **identity epoch**：丢弃旧 routing，要求重新选择，响亮失败；不能按 URL、标题或 `[selected]` 文本自动重映射。自动 retarget 的“成功率”是以错 tab 为代价的假成功。

**Counterexamples**：如果上游 id 跨重启稳定，epoch 丢弃就是多余；如果调用方每次显式传 target，proxy 的持久 selection 不是权威。KO 的 scope 已限定为“id 会重发”的协议。

## KO-05 · 新鲜度 = 内容 epoch + 零突变，不是时间

```yaml
knowledge_layer: generalized
category: STATE
abstraction: L4
value: A
epistemic_status: Validated Pattern
provenance:
  discovered_by: engineering-knowledge-miner
  supported_by: [code-analyst, test-analyst]
  contested_by: [abstraction-auditor, counterexample-hunter]
derivation:
  facts: [EK-15, EK-16, EK-17, EK-18]
  promotion_arguments:
    - "L2→L3: generation 计数器、页面 MutationObserver、uid stamp 三个机制在两个文件重复同一结构。"
    - "L3→L4: 稳定关系是'引用新鲜度是 (epoch, quiescence) 的联合谓词，不是时间戳'；时间相同也可能内容已变。"
aggregation_rule:
  rule: R2
  cluster_eks: [EK-15, EK-16, EK-17, EK-18]
  edges:
    - "causal: EK-15→EK-16→EK-17"
    - "constraint: EK-16↔EK-15"
  naming: "generation→observer→stamp 是完整链，可命名。"
  scope_expansion: "单条 EK 只解释计数器或 observer；KO 解释为何二者必须联合，且截断不能破坏 token。"
evidence:
  supporting: [EV-033, EV-034, EV-035, EV-036, EV-037]
  contradicting: []
scope:
  applies_when: "DOM/可交互页面，ref 在两次快照之间可能失效"
  does_not_apply_when: "静态内容、无 mutation 的页面；或每次操作都重新解析 selector"
confidence: high
validation:
  truth: PASS
  coverage: PASS
  abstraction: OK
  counterexample: "攻击 1：用时间戳判断新鲜度；攻击 2：只比 generation 不看 mutations；攻击 3：截断后 uid 丢 token。前两者被代码拒绝，第三项由 truncate 按行边界 + stamp 顺序覆盖，但未做 live DOM 测试。"
  counterexample_budget: {required: 3, attempted: 3, searched_paths: [timestamp, mutations, truncation]}
  blind_reconstruction: performed
flow_traceability:
  l1_facts_to_edges:
    - {l1_fact: "每次 snapshot bump generation", flow_edge: "F-S03: src/uid-freshness.ts:81-84", verified: true}
    - {l1_fact: "mutations>0 抛 STALE_REF", flow_edge: "F-S03: src/uid-freshness.ts:118-140", verified: true}
```

**Claim**：对可交互页面的引用，新鲜度必须是 `(snapshot generation, mutation count)` 的联合谓词：ref 属于当前 epoch，且页面自捕获以来零突变。时间戳、URL、标题都不能替代。**例外**：没有 `gN:` 前缀的 legacy ref 不绑定原始 epoch，只受当前页面状态约束（EK-17）。

**Counterexamples / 例外**：legacy ref 没有 generation，只受当前页面状态约束；这是兼容性例外（EK-17），KO 明确写入 scope 而不是假装所有 ref 都严格绑定。截断可能让某个 uid 不在可见文本里，但 stamp 在截断前/后顺序需由实际管线保证；本轮未做 live DOM 验证，标 PARTIAL。

## KO-06 · 生命周期需要多触发失效，不是单一 PID 检查

```yaml
knowledge_layer: generalized
category: RUNTIME
abstraction: L3
value: A
epistemic_status: Validated Pattern
provenance:
  discovered_by: failure-analyst
  supported_by: [code-analyst, test-analyst]
  contested_by: [counterexample-hunter]
derivation:
  facts: [EK-04, EK-05, EK-06, EK-07, EK-29]
  promotion_arguments:
    - "L2→L3: 四类失效（CDP target 死、MCP 关闭、idle、进程组遗留）在不同触发点重复出现，形成可复用生命周期模式。"
aggregation_rule:
  rule: R4
  cluster_eks: [EK-04, EK-05, EK-06, EK-07, EK-29]
  edges:
    - "causal: EK-04→EK-05→EK-07"
    - "mechanism: EK-06↔EK-07；EK-29↔EK-07"
    - "contrast: EK-04↔EK-06 / EK-04↔EK-29"
  naming: "四种独立失效触发共享同一 lifecycle，主题明确。"
  scope_expansion: "单条 EK 只解释一个触发；KO 解释为何 supervisor 需要多触发，并预测只查 PID 会留下的三类残留。"
evidence:
  supporting: [EV-004, EV-005, EV-006, EV-008, EV-009, EV-039, EV-046]
  contradicting: []
scope:
  applies_when: "代理进程持有下游进程/浏览器/连接，且下游可能独立死亡"
  does_not_apply_when: "下游与 supervisor 同生命周期、无独立退出路径"
confidence: high
validation:
  truth: PASS
  coverage: PASS
  abstraction: OK
  counterexample: "攻击 1：只查 PID，浏览器死仍复用；攻击 2：MCP 死后 bridge 继续 503；攻击 3：SIGTERM 后子进程残留。三者分别被 deep health、onclose、进程组 kill 覆盖。"
  counterexample_budget: {required: 3, attempted: 3, searched_paths: [pid-only, mcp-close, orphan-group]}
  blind_reconstruction: performed
flow_traceability:
  l1_facts_to_edges:
    - {l1_fact: "deep health 失败则重启", flow_edge: "F-C04: src/client.ts:473-530", verified: true}
    - {l1_fact: "MCP onclose 触发 shutdown", flow_edge: "F-C08: src/bridge.ts:1230-1250", verified: true}
```

**Claim**：supervisor 的失效触发至少应覆盖四类：下游资源死亡（deep health）、下游连接关闭（onclose）、无活动（idle）、进程组遗留（group kill）。只检查 supervisor 自己的 PID 会漏掉后三类。

## KO-07 · 跨进程边界必须显式携带 caller context

```yaml
knowledge_layer: generalized
category: ENGINEERING
abstraction: L3
value: A
epistemic_status: Validated Pattern
provenance:
  discovered_by: code-analyst
  supported_by: [test-analyst, failure-analyst]
  contested_by: [abstraction-auditor]
derivation:
  facts: [EK-27, EK-28, EK-30, EK-35]
  promotion_arguments:
    - "L2→L3: deadline、cwd、脚本 callable、console 通道四个独立上下文都在跨进程时被显式重建，形成可复用模板。"
aggregation_rule:
  rule: R4
  cluster_eks: [EK-27, EK-28, EK-30, EK-35]
  edges:
    - "constraint: EK-27↔EK-28；EK-30↔EK-28"
    - "dependency: EK-30→EK-35"
    - "contrast: EK-28↔EK-30"
  naming: "四类 caller context 都因 bridge 与 CLI 的进程边界而需要显式重建，主题可命名。"
  scope_expansion: "单条 EK 只解释一个上下文；KO 解释跨进程 wrapper 的通用模板：caller 的 cwd/deadline/callable/stdout 不能靠 bridge 默认值继承。"
evidence:
  supporting: [EV-025, EV-026, EV-038, EV-040]
  contradicting: []
scope:
  applies_when: "CLI/agent 与执行器分属不同进程，执行器有自己的 cwd/stdout/超时默认"
  does_not_apply_when: "同进程调用，或所有上下文都由执行器显式接收"
confidence: high
validation:
  truth: PASS
  coverage: PASS
  abstraction: OK
  counterexample: "攻击 1：相对输出路径按 bridge cwd 解析；攻击 2：导航超时大于 CLI deadline；攻击 3：把表达式结果当 callable 发送。三者均被代码防护，但 live 文件写入未执行。"
  counterexample_budget: {required: 3, attempted: 3, searched_paths: [bridge-cwd, deadline-inversion, expression-vs-callable]}
  blind_reconstruction: performed
flow_traceability:
  l1_facts_to_edges:
    - {l1_fact: "相对路径按 CLI cwd 解析", flow_edge: "F-D04: src/paths.ts:1-9", verified: true}
    - {l1_fact: "外层 deadline 大于内层", flow_edge: "F-D04: src/navigation-timeout.ts:60-74", verified: true}
```

**Claim**：跨进程 wrapper 必须把 caller context 显式搬运到执行侧：cwd（相对路径）、deadline（嵌套超时）、callable 形态（evaluate_script 源码）、输出通道（console.log）。执行器的默认值不是 caller 的上下文。

## KO-08 · 质量/发布声明必须由机器强制 custody

```yaml
knowledge_layer: generalized
category: WORKFLOW
abstraction: L3
value: B
epistemic_status: Validated Pattern
provenance:
  discovered_by: policy-governance-analyst
  supported_by: [test-analyst, code-analyst]
  contested_by: [counterexample-hunter]
derivation:
  facts: [EK-32, EK-33, EK-34]
  promotion_arguments:
    - "L2→L3: 测试证据、本地 lint、生成文件 guard、PR attestation 四个机制重复同一结构：声明必须由机器可检查的 artifact 所有权支撑。"
aggregation_rule:
  rule: R4
  cluster_eks: [EK-32, EK-33, EK-34]
  edges:
    - "dependency: EK-32→EK-33"
    - "causal: EK-33→EK-34"
    - "mechanism: EK-33↔EK-34"
  naming: "测试→CI→生成文件 custody 是同一治理主题的互补维度，可命名。"
  scope_expansion: "单条 EK 只解释一个 gate；KO 解释为什么发布声明需要 artifact 所有权，而不是贡献者自觉。"
evidence:
  supporting: [EV-042, EV-043, EV-044, EV-048, EV-049]
  contradicting: []
scope:
  applies_when: "有生成文件、自动 release、多人/agent 贡献的仓库"
  does_not_apply_when: "一次性脚本、无发布产物的个人实验"
confidence: medium
validation:
  truth: PASS
  coverage: PASS
  abstraction: OVER-ABSTRACTED
  counterexample: "攻击 1：手改 CHANGELOG 绕过 release 记录；攻击 2：release PR 触发永远 action_required 的 CI；攻击 3：本地 lint 用 npm/npx 绕过 pnpm 锁。三者分别被 guard、paths-ignore、airlock 测试覆盖。"
  counterexample_budget: {required: 3, attempted: 3, searched_paths: [manual-changelog, release-pr-ci, npm-lint]}
  blind_reconstruction: performed
flow_traceability:
  l1_facts_to_edges:
    - {l1_fact: "CI 只跑 build/test", flow_edge: "F-P01: .github/workflows/ci.yml:1-30", verified: true}
    - {l1_fact: "guard 检查 base...head 生成文件", flow_edge: "F-P02: .github/workflows/guard-generated-files.yml:1-60", verified: true}
```

**Claim**：在**本仓库的 release/CI 治理**中，质量与发布声明不能只靠约定；测试证据、lint、生成文件所有权、PR attestation 由机器 gate 检查。否则“发布说明”会与真实产物脱钩。该 claim 不跨项目推广。

## 聚合质量统计

- KO 总数：8；全部声明 `aggregation_rule`（100%）
- 规则分布：R1 ×1（KO-03）、R2 ×3（KO-02/04/05）、R4 ×4（KO-01/06/07/08）
- 簇规模：3–7（KO-01 7、KO-02 5、KO-03 3、KO-04 5、KO-05 4、KO-06 5、KO-07 4、KO-08 3），全部在 3–12
- `derivation.facts` 与 `aggregation_rule.cluster_eks` 完全一致（脚本校验）
- 无 L5；无 Law；无跨项目 Principle
