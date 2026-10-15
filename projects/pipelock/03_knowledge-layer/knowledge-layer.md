# 03 · Knowledge Layer（窄尖顶 · Generalized KO）— Pipelock

> 8 个 Core KO，全部声明 aggregation_rule（R1-R4）+ 簇内 EK + 解释范围扩大论证。未跨项目验证的原则标注 `Cross-project validation pending`。

## KO-01 决策证据必须由信任边界外的中介签发（外部可信证据模式）
- **aggregation_rule**: R2 因果链簇（EK-01→EK-02→EK-04→EK-08→EK-24）+ R1 机制簇（EK-01/06/07 共享 canonical-sign 机制）
- **层**: L4 Cognitive Model
- **一句话**: 被约束方（agent）不能充当自身行为权威的见证者——行为证据必须由约束方（mediator）在约束现场之外签名，供独立方验证。
- **解释范围扩大论证**: 从"Pipelock 为何要 mediator-signed receipts"扩大到"任何强制层（防火墙/审计代理/监管沙箱）如何让第三方可信其判定"；RAMPART（观测即契约）与 guardian（确定性防护）构成同一模式的跨项目实例。
- **反例**: MCP 输入扫描的"干净转发"路径（转发内容本身不再带中介签名）——只覆盖决策不覆盖转发内容（见 05-C2）。
- **状态**: Validated Pattern（Pipelock S5 实现 + RAMPART/guardian 跨项目 S8 部分）

## KO-02 fail-closed 必须以"拒绝（refusal）"而非"收窄（narrowing）"表达
- **aggregation_rule**: R3 不变量簇（EK-24/25/28/33 + EK-30/36 core floor）
- **层**: L4 Cognitive Model
- **一句话**: 安全强制中，当一个声明的前置条件无法全部满足时，正确动作是拒绝整个操作，而不是静默执行一个更窄版本——收窄会制造无法追溯的失败。
- **解释范围扩大论证**: guard 的 ErrManifestIncomplete/ErrPolicyNarrowed、airlock unknown tier、rules bundle 失败拒 reload、config 安全降级拒绝，四个独立子系统的同一不变量；对照 guardian 的 fail-closed deny 与 ai-protector 的默认拒绝。
- **反例**: sandbox 的分级降级（无 user namespace → 仅设 PROXY env）——这是**有文档的、显式的**降级而非静默收窄（EK-26）；CHANGELOG 明示严格 ARM64 below ABI 9 refuse。
- **状态**: Validated Pattern（S4/S5；跨项目 pending S8）

## KO-03 解析器的严格性差异是注入面（parser-differential smuggling）
- **aggregation_rule**: R1 机制簇（EK-03/21/27 共享"严格解析器"机制）
- **层**: L3 Pattern
- **一句话**: 当同一份数据会被不同解析器以不同方式解读（签名/显示/日志），必须对最宽松的解读路径做 fail-closed 校验（duplicate keys、unknown fields、trailing tokens、canonical keys）。
- **解释范围扩大论证**: receipt Unmarshal（三 parser-differential 面）与 MCP typed schema（canonical keys 拒绝别名/大小写变体重复键）是同一问题的两个传输面；对照 pyrit（攻击面）与 owasp-agentic-skills（威胁分类）——防御侧实例罕见。
- **状态**: Validated Pattern（S4 测试族：strict_verify_test.go / dupkey_helpers / canonical_parity_test）

## KO-04 证据链的完整性是收据价值的前提（open chain ≠ complete）
- **aggregation_rule**: R2 因果链簇（EK-11→EK-12→EK-13→EK-14）
- **层**: L3 Pattern
- **一句话**: 可验证的证据必须声明其边界状态（closed/group-complete）；"有效前缀"不是"完整链"，分片不是组。
- **解释范围扩大论证**: 从 receipt chain 语义（3.7.0 破坏性变更：VALID PREFIX exit 3）推广到任何 append-only 审计证据（区块链轻节点、SCITT、Rekor 日志）——"可验证片段"与"完整证据"必须区分。
- **反例**: PyPI pipelock-verify 0.4.1 只验分片不验组（官方文档承认其不足以证明完整组）——工具能力与证据声称必须匹配。
- **状态**: Pattern（S5 实现 + 官方文档明示边界）

## KO-05 中间层互不信任：detect ≠ decide ≠ enforce ≠ evidence
- **aggregation_rule**: R4 主题簇（EK-29/30/31/24/35/40）
- **层**: L4 Cognitive Model
- **一句话**: 安全系统中，检测、决策、执行、证据四个环节应分属独立权威，任何环节失败都应 fail-closed，且证据环节不因检测环节的乐观而乐观。
- **解释范围扩大论证**: 四层分离 + core floor（enforce 关闭也 block）+ capability manifest 显式 gate（免费层不缩水）＝"层间不信任"的完整实现；对照 agent-governance-toolkit（策略引擎层）与 RAMPART（观测层）——Pipelock 是四层齐全的实例。
- **反例**: host containment 能力声明承认"applies only to traffic routed through Pipelock; direct agent egress needs containment or network policy"——层间不信任的边界诚实声明（AGENTS.md）。
- **状态**: Validated Pattern（跨项目 pending）

## KO-06 治理值以组合形式传递，防止"只传一半"的安全静默回退
- **aggregation_rule**: R1 机制簇（EK-32/34/39）
- **层**: L3 Pattern
- **一句话**: 当两个参数共同决定信任集时，把它们封装为单一组合值（TrustPolicy{TrustedKeys, TrustEmbeddedKeys}），让"只传一个"在类型层面不可能。
- **解释范围扩大论证**: "using either without the other is always a bug"——把运行时 bug 变成编译期不可能；对照前几轮考古的 config 校验模式（thinkingbox/aprover 的 schema 校验），类型级组合是更强的约束。
- **状态**: Pattern（S4/S5；跨项目 pending）

## KO-07 攻击面分级响应：action 强度序位（block>defer>ask>redirect>strip>warn）与"无 enforcement 值永不被选中"
- **aggregation_rule**: R4 主题簇（EK-30/31/40 + config/action.go）
- **层**: L3 Pattern
- **一句话**: 多扫描器结论聚合时，取最强动作而非多数/最先；allow/forward 在强度序位中排名 0，防止降级。
- **解释范围扩大论证**: StrictestAction 与 buildDecision 的"core 优先"构成一致强度序；对照 guardian 的 deny-read→NoOverride（约束不变量）——同一"最强约束胜出"原则。
- **反例**: action_divergence.go 明确区分两个正确序位（reload 降级检测 vs 决策聚合）——同一词表两种排序，取决于问题（config/action.go 注释）。
- **状态**: Validated Pattern（S4）

## KO-08 扫描在副作用发生前完成（pre-side-effect detection）
- **aggregation_rule**: R2 因果链簇（EK-29→EK-30→EK-31）
- **层**: L3 Pattern
- **一句话**: DLP/注入扫描在 DNS/连接/写操作之前完成，被阻断的内容从未离开边界；内容重写（shield）保证注入面被消毒后才放行。
- **解释范围扩大论证**: "key never leaves the proxy"与 RAMPART 的"观测先于动作"、guardian 的 pre-exec 决策一致——检测时序是边界有效性的前提。
- **状态**: Validated Pattern（S5；跨项目 pending）

---
## 纵向知识链示例（三层贯通）
```
EK-01（签名只覆盖 canonical record —— L1 事实）
  ↓
EK-02/EK-04（外部信任锚强制；全链校验 —— L2 知识）
  ↓
KO-01（决策证据必须由信任边界外中介签发 —— L3/L4 模式）
  ↓
CK-1（Intelligence ≠ Authority 的工程实例：判断力与见证权分离 —— L4 Core）
```
