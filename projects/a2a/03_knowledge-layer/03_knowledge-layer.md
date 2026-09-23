# 03 — Knowledge Layer（Generalized Knowledge / 窄尖顶）

> 每个 KO 声明 `aggregation_rule`（R1-R4）+ 簇内 EK + 解释范围扩大论证。升维是特例不是默认——本层只保留真正跨出"本项目"的认知。

## KO 聚合规则矩阵

| KO | 规则 | 簇内 EK | 边类型 | 解释范围 |
|----|------|---------|--------|---------|
| KO-01 | R3 不变量簇 | EK-01/04/05/07 | constraint+causal | 任务生命周期设计哲学 |
| KO-02 | R4 主题簇 | EK-17/18/19/20/21 | mechanism+constraint | 协议生态演进机制 |
| KO-03 | R2 因果链簇 | EK-10→12→13→11→18→29 | causal | 安全发现-协商链 |
| KO-04 | R3 不变量簇 | EK-19/20/11/22 | constraint | Agent 安全边界不变量 |
| KO-05 | R4 主题簇 | EK-15/16/26/27 | mechanism | 异步任务交互三机制 |
| KO-06 | R4 主题簇 | EK-08/09/10 | mechanism | 多租户路由 |
| KO-07 | R1 机制簇 | EK-02/04 + MCP 文档 | mechanism | 双协议分层认知（design-intent） |

---

## KO-01 任务不可变性：可追踪性由"不可变任务 + 客户端持有版本"换取（L4 Cognitive Model）

**一句话稳定关系**：在分布式多轮协作中，把"执行单元"设为不可变对象（终态即冻结）、把"演进历史"的所有权放到客户端，用少量的重建成本（新建任务）换取输入→输出的干净映射、跨轮次可追踪与实现无歧义——**不可变性是追踪性的免费午餐，代价转移而非消除**。

- aggregation_rule: R3 不变量簇（EK-04 终态不可重启 + EK-07 客户端管版本 + EK-01 终态集合 + EK-05 引用前序任务）
- L1 事实：`[T:life §Task Immutability]`；`[P:187-208]` 终态枚举
- L2 工程知识：EK-04/EK-07/EK-01/EK-05
- L3 Pattern：**不可变工作单元 + 外部版本链**（同类：Git 提交不可变 + 分支指针演进；数据库 append-only 日志 + 快照）
- L4 Cognitive Model：追踪性来自不可变基元，版本/演进必须显式外包给一个权威所有者
- L5 Methodology：设计多轮协作协议时，把"单元不可变"与"历史谁持有"一起决策；宁可新建单元也不允许原地变异终态
- 反例攻击：若任务量大且客户端不可靠，客户端持有版本链成为单点——协议用"服务端复用一致 artifact-name"缓解（EK-07）；"新任务成本"在并行模型下被 EK-06 摊销

## KO-02 核心冻结 + 扩展治理三明治（L4 Cognitive Model）

**一句话稳定关系**：协议生态的长寿 = 核心类型冻结（禁止改结构/enum）+ 扩展层自由创新（metadata 叠加/新 RPC）+ 显式治理门（experimental→official→core）——**稳定性不靠禁止创新，靠把创新隔离在核心之外并给它晋升路径**。

- aggregation_rule: R4 主题簇（EK-20 核心稳定性铁律 + EK-17 扩展四类型 + EK-18 激活协商 + EK-19 安全铁律 + EK-21 治理生命周期）
- L1 事实：`[T:ext §Limitations]`"Extensions should place custom attributes in the metadata map"；`[T:gov]` 全生命周期
- L2 工程知识：EK-20/EK-17/EK-18/EK-19/EK-21
- L3 Pattern：**兼容性内核 + 扩展总线 + 晋升门**（同类：Kubernetes CRD+admission webhook、HTTP 头扩展、MCP tools/skills）
- L4 Cognitive Model：核心标准的演进风险集中在"类型契约变更"，把变更需求引导到受控扩展层可同时获得生态广度与核心稳定
- L5 Methodology：定义协议时先划"不可变核心集"（数据模型+枚举）与"可变扩展面"；扩展晋升核心必须有独立成熟度证据（参考实现/采用/维护承诺）而非仅投票
- 反例攻击：扩展碎片化风险——用 URI 命名空间（标识符）+ 官方/experimental 两级 + TSC 监督归档缓解（EK-21/EK-22）；"many will remain as extensions indefinitely" 是设计预期而非失败（`[T:gov]`）

## KO-03 安全发现-协商链：能力声明 → 分级披露 → 认证协商 → 按需激活（L3 Pattern）

**一句话稳定关系**：跨信任域的协作前必须先完成一条"声明→发现→披露→认证→激活"的安全链，每一步都把信任决策显式化：公开信息（Agent Card）→ 敏感信息（extended card 认证后）→ 认证方案（SecurityScheme）→ 功能激活（扩展 opt-in）。

- aggregation_rule: R2 因果链簇（EK-10→EK-13→EK-12→EK-11→EK-18，causal 链）
- L1 事实：`[T:disco]` 三发现策略；`[P:122-129]` GetExtendedAgentCard；`[P:495-517]` SecurityScheme；`[T:ext §Activation]` A2A-Extensions header
- L2 工程知识：EK-10/EK-13/EK-12/EK-11/EK-18
- L3 Pattern：**渐进式信任协商**（同类：OAuth 2.1 的声明+授权服务器发现+scope 协商；TLS 的证书链+ALPN；企业 SSO 的 metadata exchange）
- L4 Cognitive Model：跨域互操作中，信息披露粒度必须与认证强度正相关（分级披露），能力激活必须显式（opt-in 而非隐式全开）
- L5 Methodology：设计 agent 协作协议时，把"对方能发现什么/认证后能看到什么/激活什么"设为独立决策点，禁止静态 secret 内嵌（`[T:disco]` 强建议动态 out-of-band credentials）
- 反例攻击：若 Agent Card 含敏感数据且端点无认证 → 泄露（文档明确警告"Any Agent Card containing sensitive data must be protected"，`[T:disco]`）；扩展默认不激活是基线兼容的防护（EK-18）

## KO-04 Agent 安全边界不变量：任何扩展/绑定不得绕过主安全控制（L4 Cognitive Model）

**一句话稳定关系**：在 Agent 生态协议中，安全不变量必须定义为**全局绝对项**——新增能力（扩展/绑定/方法）可与主控制并存，但不得成为其旁路；这条不变量是协议可信度的底线。

- aggregation_rule: R3 不变量簇（EK-19 扩展安全铁律 + EK-20 核心冻结 + EK-11 认证方案 + EK-22 绑定要求）
- L1 事实：`[T:ext §Security]` "An extension MUST NOT provide a way to bypass the agent's primary security controls"；`[T:cpb]` "ensure the custom binding does not inadvertently bypass the agent's primary security controls"
- L2 工程知识：EK-19/EK-20/EK-11/EK-22
- L3 Pattern：**fail-closed 扩展边界**（同类：E2B 沙箱网络面 fail-closed、Tafcm 受控接口、浏览器扩展权限模型）
- L4 Cognitive Model：可扩展系统的安全 = 核心不变量 × 扩展点纪律；扩展点越多，必须把"不得绕过"写成显式规范约束而非隐含假设
- L5 Methodology：为任何 agent 协议定义扩展机制时，同时写入"同等级 authn/authz + 禁止旁路 + required 标记只用于安全核心功能"三条款
- 反例攻击：required:true 被滥用为常态会制造客户端硬依赖（文档限定"only be used for extensions fundamental to the agent's core function and security"，EK-19）；实现级风险是扩展方法漏挂中间件——规范层面只能靠审计，本 KO 覆盖规范意图而非实现验证（Epistemic: Pattern，非已验证全实现）

## KO-05 异步任务交互三机制（L3 Pattern）

**一句话稳定关系**：长运行 agent 任务需要按"客户端连接能力"分派三种通知机制——同步阻塞（默认）、SSE 流式（在线增量）、Push 通知（断连/长时），三者的终止/触发边界统一绑定任务状态机（终态/中断态）。

- aggregation_rule: R4 主题簇（EK-15/16/26/27 覆盖同步/流式/推送/负载四维度）
- L1 事实：`[P:155-160]` return_immediately；`[P:33-42, 74-82]` streaming；`[P:469-485]` push config；`[P:790-803]` StreamResponse
- L2 工程知识：EK-26/EK-15/EK-16/EK-27
- L3 Pattern：**连接能力适配的通知分层**（同类：REST 轮询 vs SSE vs WebSocket；MQTT QoS 分级；手机推送 + 拉取兜底）
- L4 Cognitive Model：异步系统的通知机制应正交于任务状态机——状态是唯一事实源，通知只是状态变化的投影通道
- L5 Methodology：设计长任务协议时先定义状态机（含中断态），再按客户端形态提供同步/流式/推送投影；推送必须支持"验真→拉全量"的补全模式（EK-16 GetTask 兜底）
- 反例攻击：流式连接断线 = 状态丢失 → SubscribeToTask 重订阅（EK-15）；推送验真失败 → 拒收 + GetTask 轮询兜底（EK-14/EK-16）——双通道互补而非互斥

## KO-06 多租户路由与 tenant 契约（L3 Pattern）

**一句话稳定关系**：单端点服务多 agent 时，路由键的选择是运维自由（URL/认证头/body），但**契约义务在客户端**——接口声明了什么（tenant）客户端必须原样 echo，未声明则必须省略；协议不定义路由语义，只定义义务边界。

- aggregation_rule: R4 主题簇（EK-08/09/10 覆盖路由义务、三方案、接口声明）
- L1 事实：`[T:multi]` 三方案全文 + 客户端 MUST/MUST NOT 规则（§8.3.2）
- L2 工程知识：EK-08/EK-09/EK-10
- L3 Pattern：**不透明路由键 + 客户端回显义务**（同类：Host 头、SNI、租户 id 注入中间件）
- L4 Cognitive Model：平台层路由的自由度与协议层的义务确定性可以共存——把语义留在实现、把义务写进规范
- L5 Methodology：设计多租户协议时：服务器自由选路由方案；协议只规定"声明则 echo、未声明则省略"，避免规范过度指定路由格式
- 反例攻击：客户端 echo 错误 tenant → 请求路由到错误 agent；协议靠"接口级声明"让客户端无歧义取键（EK-10），但无服务端校验机制——Epistemic 标"协议意图"非已验证行为

## KO-07 A2A/MCP 分层认知：深度归工具、广度归协作（L4 Cognitive Model，design-intent 标注）

**一句话稳定关系**：Agent 系统的能力边界由两个正交维度构成——MCP 垂直加深单个 agent（工具/资源），A2A 水平连接跨边界 agent（协作/协商/任务共享）；**"一个 agent 多能"与"多 agent 协作"是不同层次的互操作**。

- aggregation_rule: R1 机制簇（EK-02 双轨 + 文档设计意图；跨 ≥2 文档：a2a-and-mcp + specification）
- L1 事实：`[T:a2a-mcp]` "MCP is vertical… A2A is horizontal"；"Used together, MCP gives each agent depth, and A2A gives your system reach"；`[T:a2a-mcp §Representing A2A Agents as MCP Resources]`
- L2 工程知识：EK-02（Message/Task 双轨）作为协作面的实现基础
- L3 Pattern：**垂直工具面 + 水平协作面**（同类：单体微服务内部 RPC vs 跨域 B2B API；浏览器扩展 API vs 跨域 postMessage）
- L4 Cognitive Model：互操作分层——工具调用是"能力使用"，agent 协作是"能力伙伴化"；协议设计应尊重这个分层，避免把协作协议退化成工具调用协议
- L5 Methodology：搭建 agentic 系统时同时规划两层：内部工具用 MCP（或等价），跨 agent 协作用对等协议；A2A 的 skills 可投影为 MCP 资源，但状态化协作不可压缩为工具调用
- 反例攻击：stateful 协作（多轮澄清/中断态）用 stateless 工具调用表达 → 丢失上下文与追踪（EK-02 双轨的动机）；本 KO 证据为**文档设计意图**（`[T:a2a-mcp]`）非运行时验证 → Epistemic 标注 design-intent，Cross-project validation pending

---

## 五层阶梯递进示例（纵向知识链，KO-01 展开）

```
L1 工程事实：TaskState 9 枚举（4 终态+2 中断+2 进行+UNSPECIFIED）；终态任务不可重启；细化必须同 contextId 新建任务 —— [P:187-208][T:life]
    ↓
L2 工程知识：终态不可变性 + 客户端管理 artifact 版本 + 服务端复用 artifact-name（EK-04/EK-07/EK-05）
    ↓
L3 Pattern：不可变工作单元 + 外部版本链（Git 提交 / append-only 日志 同类对照）
    ↓
L4 Cognitive Model：追踪性来自不可变基元，版本/演进必须显式外包给一个权威所有者
    ↓
L5 Methodology：设计多轮协作协议时把"单元不可变"与"历史谁持有"一起决策；禁止原地变异终态
```

## 层间统计
- Generalized KO：7（L3×3、L4×4，符合"7~12 窄尖顶"下限；小规范库按比例缩放）
- 全部 KO 可回溯 EK（无悬空升维 ✓）
- aggregation_rule 覆盖率 100% ✓
