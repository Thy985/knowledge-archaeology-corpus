# 03 Knowledge Layer — Generalized KO（KO-01…KO-10）

> 口径：KO 只从 EK 图上的簇聚合（R1 机制簇 / R2 因果链簇 / R3 不变量簇 / R4 主题簇），每条声明 `aggregation_rule`、簇内 EK、解释范围扩大论证、反例。L3 以上才允许称 Pattern/Model；未跨项目验证的统一标 `Cross-project validation pending`，不写成 Principle/Law。
> 本层 10 个 KO，其中 7 个 L3、3 个 L4（均带 cross-project pending）。没有任何 KO 被标为 L5 方法论——AMG 是单项目证据，本轮不产出“普遍行动准则”。

## KO-01 授权判断必须先于副作用与内容检查
- **aggregation_rule**：R2 因果链簇（EK-01→EK-02/03→EK-04→EK-21→EK-22）
- **簇内 EK**：EK-01、EK-02、EK-03、EK-04、EK-21、EK-22
- **L1 事实**：`decide()` 五阶段短路；read gate 在 `_read_impl()` 之前；被拒 write 不跑 detectors、不写 store、不产生快照；被拒 read 不暴露 key 是否存在。
- **L2 工程知识**：因为拒绝如果发生在副作用之后，会（a）留下部分写入/快照，（b）让未授权者通过异常/延迟差异探测 key 存在性，（c）让 detector 处理未授权内容，扩大攻击面。
- **L3 模式**：**Pre-authorization boundary**——在安全关键管线中，授权判断应作为 pre-condition，而不是与业务逻辑并行的 filter；拒绝路径必须是“零副作用 + 统一外观”。
- **L4 候选**：安全边界的位置决定它能否同时提供 confidentiality 与 integrity；边界越靠前，能保护的资产越多，但也会牺牲错误信息粒度。
- **L5 行动**：不输出。单项目证据不足以写成普遍行动准则。
- **反例**：`admins` 的 admin 操作在 key rule 之前短路（EK-03）——同一“先于”原则，但顺序相反（admin 先于 rule）。这不否定模式，说明“先于”的对象是“副作用”，不是“所有规则”。
- **Epistemic**：L3 Pattern · AMG-originated · Strongly evidenced（S4）· Cross-project validation pending。
- **可回溯**：KO→EK-01/02/03/04/21/22→`policies/access.py:445-544`、`guard.py:408-455, 950-1004`、`tests/test_access_control.py`。

## KO-02 无法执行的内容处置必须变成显式 finding/withhold
- **aggregation_rule**：R3 不变量簇（EK-27、EK-28、EK-30、EK-36、EK-37 汇聚到“不得把未处置内容交出去/存下去”）
- **簇内 EK**：EK-27、EK-28、EK-30、EK-36、EK-37
- **L1 事实**：REDACT 无 redactor → write quarantine / read `_WITHHELD`；integrity mismatch → CRITICAL BLOCK；深度超限 → `size_anomaly` HIGH；quarantine 不写 store、不覆盖旧值。
- **L2 工程知识**：安全管线不可能对所有内容都有“正确的变换”；当变换不可执行时，系统必须选择“拒绝/隔离/保留”而不是“放行原件”。AMG 的选择是：write 隔离、read 扣留、integrity 拒绝、depth 追加 finding。
- **L3 模式**：**No-passthrough-on-unhandled**——若策略要求脱敏但当前 detector 无能力，默认不返回原值；把“不能处理”转成“不可输出”的显式状态。
- **L4 候选**：安全默认不是“尽量返回可用数据”，而是“无法证明已处置就不返回”。
- **反例**：`SensitiveDataDetector.redact()` 对容器返回字符串（EK-28）——它“成功脱敏”了，但破坏了结构；说明“不 passthrough”与“保持语义”是两个独立目标，不能用一个成功掩盖另一个失败。
- **Epistemic**：L3 Pattern · Strongly evidenced（S4）· Cross-project pending。
- **可回溯**：KO→EK-27/28/30/36/37→`guard.py:838-900, 1017-1058, 1423-1470`、`detectors/injection.py`、`tests/test_redaction_fail_closed.py`。

## KO-03 Detector 失败是 fail-open，但必须可见
- **aggregation_rule**：R4 主题簇（EK-25、EK-26、EK-32、EK-38 围绕“检测器异常/缺失时的可用性”）
- **簇内 EK**：EK-25、EK-26、EK-32、EK-38
- **L1 事实**：单个 detector 抛异常 → 丢失 verdict + LOW ALLOW 运维事件 + continue；默认只注册 7 个；`PrivilegeEscalation` 等未默认启用；events 无上限。
- **L2 工程知识**：detector 是用户可扩展代码，若异常直接阻断所有记忆操作，会把“检测器 bug”放大成“agent 全面不可用”；因此选择 fail-open。代价是“检测器静默失效”的风险，必须靠事件可见性补偿。
- **L3 模式**：**Fail-open with observable degradation**——安全组件的失败不阻断主流程，但必须产生可观测的降级信号，并且默认套件要明确列出“哪些检测未启用”。
- **L4 候选**：安全性与可用性的取舍不能只体现在代码分支，必须体现在默认配置与可观测性上；否则“fail-open”会退化成“无人知道它没工作”。
- **反例**：EK-25 的 LOW ALLOW 事件没有强制告警/阈值；如果没人消费 events，可见性等于零。EK-38 的 events 无上限，长期运行也可能丢历史/占内存。
- **Epistemic**：L3 Pattern · AMG-originated · Evidence S4（测试覆盖异常路径）+ S3（默认套件）· Cross-project pending。
- **可回溯**：KO→EK-25/26/32/38→`guard.py:1333-1422`、`guard.py:150-163`、`events.py`。

## KO-04 信任晋升是图 + 门禁，不是布尔开关
- **aggregation_rule**：R2 因果链簇（EK-04→EK-05→EK-34→EK-35→EK-41，加上 EK-54 的审计 caveat）
- **簇内 EK**：EK-04、EK-05、EK-34、EK-35、EK-41、EK-54
- **L1 事实**：promotion graph 只允许 3 条默认边；`candidate→verified` 要 `verified=True`；gated class 还要 `class_writers`；rollback 若改 gated class 被拒；`verified_by` 只是 metadata。
- **L2 工程知识**：单一布尔（`verified=True`）不足以表达“谁在什么权限下批准了什么”；AMG 用 graph 限制“哪些类可以到哪些类”，用 class gate 限制“谁能改高信任类”，用 rollback check 防止“用快照绕过门禁”。但 `verified_by` 未认证，说明“审计归属”仍是弱环。
- **L3 模式**：**Trust transition as a constrained graph**——信任等级之间的迁移应由（源类、目标类、验证要求、授权主体）四元组约束，而不是一个可由调用方自报的布尔。
- **L4 候选**：信任不是二值属性，而是“来源类别 + 迁移路径 + 授权主体”的联合状态；任何单点布尔都会在审计/回滚/委派场景失效。
- **反例**：EK-54 证明 `verified_by` 可伪造；EK-53 证明 target 归一化不一致。两者都削弱“图 + 门禁”的完整性，但不否定图本身。
- **Epistemic**：L3 Pattern · Strongly evidenced（S4）· Cross-project pending。
- **可回溯**：KO→EK-04/05/34/35/41/54→`classification.py:42-63`、`policies/access.py:394-544`、`guard.py:556-671, 1216-1332`。

## KO-05 作用域边界的强度等于最弱的等价路径
- **aggregation_rule**：R1 机制簇（EK-42、EK-43、EK-44、EK-45、EK-55 共享“绕过主路径”机制，跨 llamaindex/其他适配器/raw guard 三个独立位置）
- **簇内 EK**：EK-42、EK-43、EK-44、EK-45、EK-55
- **L1 事实**：LlamaIndex `get_messages` 直读底层；测试默认接受该读旁路；适配器公开 `.guard`；LangChain/CrewAI/Agno 各有内部 store 读；`list_snapshots` 等 raw-guard 接口未 gate。
- **L2 工程知识**：per-agent ACL 只在 `guard.read/write` 路径上执行；任何“等价 API”（`get_messages`、`get_keys`、`_store.get`、raw guard）如果直接触达 store，就会绕过 gate。边界不是一个点，而是一组等价入口的集合。
- **L3 模式**：**Scope closure over equivalent paths**——安全作用域必须覆盖所有能产生同等效果的入口，而不是只覆盖“主 API”；适配器的便利方法、内部属性、审计接口都属于等价路径。
- **L4 候选**：一个系统的“权限边界”应以其最弱等价路径定义，而不是以设计文档中的主路径定义。
- **反例**：LangChain `messages` 走 `guard.read`（EK-45），说明并非所有适配器都绕过；CrewAI `write` 的底层写入不在 guard 事务内，但读路径仍受控。边界是“部分闭合”，不是“全开放”。
- **Epistemic**：L3 Pattern · AMG-originated · Strongly evidenced（S5 复现 + 代码）· **Cross-project validation pending**（尚未在第二个 memory 系统验证“最弱路径”定律）。
- **可回溯**：KO→EK-42/43/44/45/55→`integrations/llamaindex.py:63-93`、`tests/test_integration_llamaindex.py:47-78`、`guard.py:529-554, 1212-1214`。

## KO-06 配置无法表达意图时必须 fail closed
- **aggregation_rule**：R3 不变量簇（EK-12、EK-13、EK-06 汇聚到“不得把未生效的权限配置当成已生效”）
- **簇内 EK**：EK-06、EK-12、EK-13
- **L1 事实**：0.3 静默忽略 `agents`/`writers` 等字段；0.4 `_check_fields()` 对其 ValueError；错误信息指向 `Policy.with_access`；无 access 时整体不检查。
- **L2 工程知识**：配置格式不支持某个安全字段时，“忽略”会让部署方以为已收紧，实际是全开；这比报错更危险。AMG 选择“无法表达 → 拒绝启动/拒绝加载”，把误配置变成显式失败。
- **L3 模式**：**Fail-closed configuration parsing**——安全配置中出现无法被当前版本执行的字段时，必须报错，不得静默降级为“无此配置”。
- **L4 候选**：安全配置的“向后兼容”不能以“静默忽略”实现；兼容性应表现为“旧字段报错并给出迁移路径”，而不是“旧字段无声失效”。
- **反例**：无 access 的 0.3 兼容路径（EK-06）仍然是“全开”，这是有意的 backcompat，不是配置解析失败；两者边界是“是否声明了 access 意图”。
- **Epistemic**：L3 Pattern · Strongly evidenced（S4，YAML guard 测试）· Cross-project pending。
- **可回溯**：KO→EK-06/12/13→`policies/policy.py:455-499`、`tests/test_policy_yaml_guard.py:24-49`。

## KO-07 执行权与内省权应分离为不同能力面
- **aggregation_rule**：R4 主题簇（EK-14、EK-15、EK-16、EK-55 围绕“handle vs raw guard”）
- **簇内 EK**：EK-14、EK-15、EK-16、EK-55
- **L1 事实**：`AgentHandle` 只暴露业务方法、拒绝 principal 覆写、不暴露 events/quarantine/snapshots/classification；raw guard 暴露这些且多数不 gate；但 `AgentHandle._guard` 仍可被 Python 访问，适配器又公开 `.guard`。
- **L2 工程知识**：在进程内，“不给 agent raw guard”是能力裁剪（capability trimming），不是认证；只要对象引用可达，Python 无法阻止访问。AMG 的 handle 设计降低了误用概率，但没有形成强制隔离。
- **L3 模式**：**Capability-surface separation**——把“能执行写/读”的能力与“能查看全量审计/快照/隔离区”的能力拆成不同对象；执行者只拿前者。
- **L4 候选**：安全边界可以由“对象能力”而非“身份检查”构成；但能力裁剪只在引用不可达时成立，一旦共享对象图，边界退化为约定。
- **反例**：EK-44 的 `.guard` 公开属性使 handle 的能力裁剪失效；EK-55 的 raw guard 接口未 gate。两者说明“分离”是设计意图，不是运行时保证。
- **Epistemic**：L3 Pattern + **L4 Candidate**（`Cross-project validation pending`）· Evidence S3/S5。
- **可回溯**：KO→EK-14/15/16/55→`identity.py:568-648`、`guard.py:529-554, 1212-1214`、`integrations/llamaindex.py:63-64`。

## KO-08 可观测性接口本身是权限面
- **aggregation_rule**：R1 机制簇（EK-38、EK-39、EK-40、EK-55 共享“审计数据可读”机制，跨 events/snapshots/raw-guard 三处）
- **簇内 EK**：EK-38、EK-39、EK-40、EK-55
- **L1 事实**：events 记录 principal/source/metadata，无上限；snapshots 含全量 store data，有界 50，digest 不校验；`list_snapshots`/`quarantine`/`events` 未 gate；只有 `snapshot()` 本身受 admin gate。
- **L2 工程知识**：安全系统的审计/取证数据往往包含原始敏感内容（quarantine 原值、snapshot data、event metadata）；如果这些接口对执行 agent 可见，等于给了一个绕过写/读门的“侧信道”。AMG 把它们留在 raw guard，但 raw guard 可通过适配器/`_guard` 触达。
- **L3 模式**：**Observability is an authority surface**——事件、隔离区、快照、分类注册表都应视为与读写同级的权限资产；只读审计也需要授权。
- **L4 候选**：安全系统的“只读”接口不是低权限；它能泄露被保护的内容与策略结构。
- **反例**：`AgentHandle` 明确不暴露这些（EK-14），说明设计者知道该风险；但 `list_snapshots` 等仍未 gate，说明“知道”未转化为“执行”。
- **Epistemic**：L3 Pattern · Strongly evidenced（S3/S5）· Cross-project pending。
- **可回溯**：KO→EK-38/39/40/55→`guard.py:529-554, 1212-1214`、`storage/snapshots.py:14-90`。

## KO-09 文档声明不等于实现事实
- **aggregation_rule**：R4 主题簇（EK-48、EK-49、EK-50、EK-52 围绕“声明 vs 证据”）
- **簇内 EK**：EK-48、EK-49、EK-50、EK-52
- **L1 事实**：design.md 的 detector 数量、Classification 表、read 流程与实现不一致；README 的 92.5%/62 µs 是 0.2.2 的 55-case 自报；10 个 xfail 明确未实现；504 tests 通过不等于这些攻击族已覆盖。
- **L2 工程知识**：文档描述的是“设计意图/历史指标”，不是“当前实现”。安全审计若把 README/design 当事实，会高估覆盖；必须用代码、测试、运行时探针分别取证。
- **L3 模式**：**Intent–implementation–test triangulation**——同一安全属性至少要有“文档声明 / 代码实现 / 测试或运行时证据”三路；任一路缺失就降级为 Observation/Hypothesis。
- **L4 候选**：安全声明应按证据强度分级，而不是按文档语气分级；版本与语料必须随指标一起声明。
- **反例**：EK-52 的 504 passed 是强证据，但它只覆盖测试集，不覆盖 xfail 的 10 族；EK-50 的 benchmark 是强证据但 pin 在旧版本。两者都说明“有证据”≠“覆盖声明范围”。
- **Epistemic**：L3 Pattern + **L4 Candidate**（`Cross-project validation pending`）· Evidence S4/S5。
- **可回溯**：KO→EK-48/49/50/52→`docs/architecture/design.md`、`README.md:114-122`、`docs/compliance-mapping.md:6,25-34`、pytest 输出。

## KO-10 基准数字必须带版本、语料与作者身份
- **aggregation_rule**：R3 不变量簇（EK-50、EK-51、EK-52 汇聚到“指标可复现/可归因”）
- **簇内 EK**：EK-50、EK-51、EK-52
- **L1 事实**：55 cases（40 attack + 15 benign），recall 92.5%、precision 100%、F1 0.961、median 62 µs，全部 pin 在 AMG 0.2.2（2026-05-02）；0.4.0 未复跑；benchmark 由项目作者编写并自评；CI matrix 3.9–3.14 但本轮只跑 3.12。
- **L2 工程知识**：安全指标不是“一个数”，而是“版本 + 语料 + 作者 + 运行环境 + 复现命令”的联合声明。缺任一维度，数字就不能直接用于 0.4.0 的能力判断。
- **L3 模式**：**Metric provenance binding**——引用检测率/延迟时必须同时引用版本、语料规模、作者身份与复现命令；跨版本比较必须重跑。
- **L4 候选**：在没有独立第三方语料的情况下，自报指标只能证明“对已知攻击形状有效”，不能证明“对自适应 adversary 有效”。
- **反例**：AMG 自己在 `docs/compliance-mapping.md:32` 明确写了这一限制（“not a measure of performance against an adaptive adversary”），说明项目方也承认边界；这是正面证据，不是反例。
- **Epistemic**：L3 Pattern · Strongly evidenced（文档自述 + 版本 pin）· Cross-project pending。
- **可回溯**：KO→EK-50/51/52→`README.md:114-122`、`docs/compliance-mapping.md:6,25-34`、`benchmarks/results/benchmark_report.md`、pytest 输出。

## KO 聚合矩阵
| KO | 规则 | 簇内 EK | 层 | Epistemic |
|---|---|---|---|---|
| KO-01 | R2 | EK-01/02/03/04/21/22 | L3 | Pattern, S4, cross-project pending |
| KO-02 | R3 | EK-27/28/30/36/37 | L3 | Pattern, S4 |
| KO-03 | R4 | EK-25/26/32/38 | L3 | Pattern, S4/S3 |
| KO-04 | R2 | EK-04/05/34/35/41/54 | L3 | Pattern, S4 |
| KO-05 | R1 | EK-42/43/44/45/55 | L3 | Pattern, S5, cross-project pending |
| KO-06 | R3 | EK-06/12/13 | L3 | Pattern, S4 |
| KO-07 | R4 | EK-14/15/16/55 | L3 + L4 cand. | Pattern/Model candidate |
| KO-08 | R1 | EK-38/39/40/55 | L3 | Pattern, S3/S5 |
| KO-09 | R4 | EK-48/49/50/52 | L3 + L4 cand. | Pattern/Model candidate |
| KO-10 | R3 | EK-50/51/52 | L3 | Pattern, S4 |

## 未升维声明
- 以下 EK 明确**不**升维，保留在 Engineering Knowledge 层：EK-07/08/09/10（selector 细节）、EK-11（dataclass 复制）、EK-17/18/19（contextvars 实现细节）、EK-20（锁）、EK-23/24（深拷贝实现）、EK-29/30/31（具体 detector 规则）、EK-33/34/35（integrity 具体配置）、EK-37（quarantine 存储）、EK-39/40（snapshot 结构）、EK-46/47（server 配置）、EK-53（API 不一致）、EK-54（verified_by）。
- 理由：它们是“具体机制/边界/实现”，回答“这个项目怎么工作”，但不足以解释更广范围；强行升维会制造“漂亮但无据”的 KO。
