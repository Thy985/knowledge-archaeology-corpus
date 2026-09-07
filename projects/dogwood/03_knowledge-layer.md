# 03 · Knowledge Layer — Dogwood（Generalized Knowledge，窄尖顶）

> v3.1 规范：每个 KO 声明 `aggregation_rule`（R1-R4）+ 簇内 EK。单项目证据标注 `Dogwood-originated · Cross-project validation pending`。

## 03.0 KO 聚合规则矩阵

| KO | aggregation_rule | 簇内 EK | 解释范围扩大 |
|----|-----------------|---------|--------------|
| KO-01 | R4 主题簇 | EK-01/02/03/04/05（temporal 族） | 从"时间语法"扩到"agent 治理需要过去维度" |
| KO-02 | R2 因果链簇 | EK-06→07→08→21（lowering 链） | 从"编译到 Cedar"扩到"扩展语义由编译期物化" |
| KO-03 | R1 机制簇 | EK-09/10/12/27（provider 注入面，跨 provider/authorize 独立实现） | 从"provider 出错"扩到"注入面安全设计" |
| KO-04 | R3 不变量簇 | EK-09/14/11/26（约束汇聚：决策可解释、隔离不变量、边界明示） | 从参考实现扩到"诚实边界与语义/实现分层" |
| KO-05 | R1 机制簇 | EK-17/18/19/20（治理机制，跨 AGENTS.md/CLI/docs 独立实现） | 从"agent 工作流"扩到"语言项目自治理" |
| KO-06 | R4 主题簇 | EK-15/16/22/28（测试文化） | 从"反例测试"扩到"测试即契约文档" |

## 03.1 KO（L3 Pattern）

### KO-01 治理需要"过去"维度：时间条件是 agent 权限的关键扩展
- **内容**：一次性 permit/deny 不足以治理 agent——需要"过去发生了什么"（since/formerly/once/窗口聚合）进决策；实现为事件 schema（decision vs history points）+ max_window 回看上限 + 范围限制静态分析。
- **回溯**：EK-01/02/03/04/05。
- **认知状态**：Pattern（L3）· 与 omnigent ASK 审批（历史状态）同域 · Cross-project validation pending。

### KO-02 扩展语义由编译期物化，而非新造运行时
- **内容**：temporal/provider 扩展不新造求值器，而是 lower 成目标语言（Cedar）的 `context.*` 槽 + schema 增广——采纳成本低、生态复用；两阶段 lowering + distincter 支持增量可组合（不同来源子策略独立 lower 再合并）。
- **回溯**：EK-06/07/08/21。
- **认知状态**：Pattern（L3）· 与 evolver 协议 prompt 化、omnigent 双评估的"编译/物化"族同域 · Cross-project validation pending。

### KO-03 注入面安全：provider 脚本与事件字段都是不可信输入
- **内容**：provider（Rhai）默认无沙箱限制是风险，参考实现给确定性引擎（new_raw + 白名单 + max_operations 后盾）；事件字段不得构造 URL authority；契约层把 provider 错误定为 UNDEFINED BEHAVIOR、实现层选 deny-on-error（infallible，错误进 diagnostics）。
- **回溯**：EK-09/10/12/27。
- **认知状态**：Pattern（L3）· 与 omnigent CredentialProxy、rampart 沙箱同域 · Cross-project validation pending。

### KO-04 参考实现的诚实边界 + 语义/实现分层
- **内容**：参考解释器把生产缺口写成 README 限制清单（审计日志缺失/多租户无隔离/错误泄露/Rhai 无默认限制）而非静默；语义契约与实现选择在测试里显式分层（"策略不得依赖任一侧"）；pin 分区是重写 pass 而非存储隔离——文档明示。
- **回溯**：EK-09/11/14/26。
- **认知状态**：Pattern（L3）· 与 omnigent OBSERVABILITY 诚实审计同族（corpus 跨项目印证中）· Cross-project validation pending。

### KO-05 语言项目自治理：agent 工作流 skill + 强制 CLI 门 + docs-as-tests
- **内容**：项目用 .claude/skills（生命周期单源真相）+ 每步强制 `dogwood` CLI 验证门 + docs 示例 = CLI 检查 bundle（doc 编译失败 = build failure）+ 单 crate 编译器强制封装——从"代码正确"到"文档不漂移"的全链治理。
- **回溯**：EK-17/18/19/20。
- **认知状态**：Pattern（L3）· 与 knowledge-archaeology-skill 的 skill 治理、skillfortify 同域 · Cross-project validation pending。

### KO-06 测试即契约文档
- **内容**：边界测试（boundary_* 拒绝路径）、对抗测试（cedarify_adversarial）、期望失败目录、语义边界写入测试 docstring（fail_closed 既测行为又写清契约/实现分界）——测试同时是规格。
- **回溯**：EK-15/16/22/28。
- **认知状态**：Pattern（L3）· 与 verifier-hub/skillfortify 的验证文化同域 · Cross-project validation pending。

## 03.2 Cognitive Models（L4）

### CM-01 时间维度是状态化治理的必需品
- **内容**：**对会积累状态的执行者（agent），权限判定必须能引用其过去——一次性检查无法表达"自登录以来/窗口内 N 次"这类治理约束。**
- **回溯**：KO-01（EK-01~05）。
- **认知状态**：Cognitive Model（L4）· 单项目强证据。

### CM-02 扩展语义物化进既有语言，而非发明新运行时
- **内容**：**新治理语义的采纳成本由"是否复用目标生态"决定——lowering 到既有语言（Cedar）+ 编译期物化扩展字段，是最低阻力路径。**
- **回溯**：KO-02（EK-06~08）。
- **认知状态**：Cognitive Model（L4）· 单项目强证据。

### CM-03 语义契约与实现选择必须分离
- **内容**：**语言契约（什么保证成立）与参考实现（当前怎么处理）是两个信任层——策略/用户若依赖实现层行为，实现一改就崩；必须显式分层并写进测试。**
- **回溯**：KO-04（EK-09/28）。
- **认知状态**：Cognitive Model（L4）· 单项目强证据。

### CM-04 注入面宽度 = 治理系统风险面
- **内容**：**治理系统（guardrail/policy）本身拥有的每个注入面（脚本引擎、事件字段、错误输出）都是攻击面——"治理者"与"被治理者"同等不可信。**
- **回溯**：KO-03（EK-09/10/27）。
- **认知状态**：Cognitive Model（L4）· 与 EP-002 同域。

## 03.3 Methodologies（L5）

### M-01 治理语言先有形式化规格与边界测试，再有生产实现
- **内容**：**agent 治理语言应：formal spec（guide 08）+ 契约/实现分层测试 + 生产边界清单——先证明语义，再谈部署。**
- **回溯**：KO-04/06。
- **认知状态**：Methodology（L5）· Dogwood 验证。

### M-02 从 MCP tools/list 生成 action schema
- **内容**：**agent 工具治理的 schema 可从 MCP tools/list manifest 自动生成——"每工具一 action + context 布局"无需手写。**
- **回溯**：KO-05（EK-19）。
- **认知状态**：Methodology（L5）· Dogwood 验证。

### M-03 文档即测试（docs-as-tests）
- **内容**：**让每个文档示例成为可执行、可验证的 bundle（policy+schema+trace），文档编译失败 = 构建失败——消灭文档漂移。**
- **回溯**：KO-05（EK-18）。
- **认知状态**：Methodology（L5）· Dogwood 验证。

## 03.4 三层配比

| 层 | 目标 | 实际 |
|----|------|------|
| Facts/Evidence | 100+ | ~35（01 层 + EK 证据） |
| Engineering Knowledge | 40~60 | 28（小项目按比例缩放） |
| KO（L3） | 7~12 | 6 |
| CM（L4） | 少量 | 4 |
| M（L5） | 少量 | 3 |
