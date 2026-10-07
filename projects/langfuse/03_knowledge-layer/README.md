# Knowledge Layer — langfuse

> Generalized KO 层（L3/L4/L5）。每个 KO 声明 aggregation_rule（R1-R4）+ 簇内 EK + 解释范围扩大论证。Epistemic status 诚实标注。

## KO 聚合规则矩阵

| KO | 层级 | 聚合规则 | 簇内 EK | 解释范围 |
|---|---|---|---|---|
| KO-01 | L4 | R2 因果链簇 | EK-01/02/03/04/06/07/08 | 摄取管道端到端机制 |
| KO-02 | L4 | R1 机制簇 | EK-09/10/11/12/13/14 | 外部语义归一化适配模式 |
| KO-03 | L4 | R2 因果链簇 | EK-15/16/17/18/21/22/24 | eval 规格→执行→治理闭环 |
| KO-04 | L4 | R3 不变量簇 | EK-19/20/26 + EK-22 | "可暂停的自动化"治理不变量 |
| KO-05 | L4 | R4 主题簇 | EK-27/28/29/30/31/32 | 队列化异步平台架构模型 |
| KO-06 | L4 | R3 不变量簇 | EK-40/41/42/43/44 | 供应链信任边界不变量 |
| KO-07 | L3 | R4 主题簇 | EK-33/34/35/36/37/38/39 | 双存储（元数据+事件）分层模型 |
| KO-08 | L5 | R3 不变量簇 | KO-01~07 跨簇 | 可观测性平台方法论 |

---

## KO-01｜事件化摄取管道（因果链）｜L4
**Cognitive Model**：LLM 观测数据的可靠摄取 = 规格化事件 + 确定性采样 + 时间感知延迟 + 溢出通道 + 失败可见。
- aggregation_rule: R2 因果链簇——EK-01(事件规格)→EK-02(批处理)→EK-03(延迟)→EK-06(采样)→EK-05(模型匹配)→EK-33(落库)，形成"输入→校验→决策→归一化→存储"完整因果链
- 证据：types.ts:319 / processEventBatch.ts:59 / sampling.ts:30 / modelMatch.ts:31 / eventsTable.ts:1
- 反例攻击：延迟仅防跨日乱序，同日乱序仍需事件时间戳兜底——模型解释范围 = "日期边界 + 载荷边界"，不承诺全序
- Epistemic: **Validated Pattern**（langfuse 内验证；跨项目验证 pending）

## KO-02｜外部语义归一化适配器（机制簇）｜L4
**Cognitive Model**：异构观测来源（OTel/日志/专有 SDK）进入统一内部模型 = 注册表优先适配 + 词表保守映射 + 未识别值显式降级（不掩盖）。
- aggregation_rule: R1 机制簇——EK-09/10/11/12/13/14 共享"外部语义→内部模型"机制，跨 ingestion/otel 两个独立子系统出现
- 证据：ObservationTypeMapper.ts:1-60 / OtelIngestionProcessor.ts:29-56 / attributes.ts:1-50
- 反例攻击：SimpleAttributeMapper 仅处理单属性键映射，复合判定（scope/spanName 参与）的 mapper 是扩展点而非默认——模型不承诺"全自动正确分类"
- Epistemic: **Validated Pattern**（适配器注册表模式在 OTel 适配层完整实现）

## KO-03｜eval 规格→执行→治理闭环（因果链）｜L4
**Cognitive Model**：规模化 LLM 评估 = 结构化问题规格（非自由文本）+ 队列化执行 + 输出 schema 校验 + 失败自动阻断。
- aggregation_rule: R2 因果链簇——EK-15(DecisionModel)→EK-16(执行)→EK-17(队列)→EK-18(编排)→EK-21(judge)→EK-24(输出校验)→EK-22(错误阻断)，形成"规格→执行→校验→治理"闭环
- 证据：decisionModel.ts:1-80 / evalQueue.ts:29-345 / llmEvaluatorExecution.ts:1-70 / evalConfigBlocking.ts:1-50
- 反例攻击：DecisionModel 限定 CHOICE/SCORE/NOUL 三类，自由格式 judge（如"返回 JSON"）走另一路径 llmEvaluatorExecution——模型描述的是结构化路径，非全部 eval 形态
- Epistemic: **Validated Pattern**（langfuse 内验证）

## KO-04｜可暂停的自动化（治理不变量）｜L4
**Cognitive Model**：自动化系统的"暂停"必须是第一类治理对象——运行开关（LIVE/MANUAL）与健康阻断（blocked）分层，手动模式可绕过开关但不可绕过健康门。
- aggregation_rule: R3 不变量簇——EK-19/20/26 经 constraint 边汇聚到"eval 规则可被阻断且阻断不可被模式绕过"不变量；EK-22 强化（错误→自动阻断）
- 证据：evalConfigBlocking.ts:41-50（"Manual batch runs bypass only the live-traffic toggle, not blocked rules"）
- 反例攻击：blockedAt 仅是软门（规则仍存在，可 reactivate）——模型是"可恢复暂停"非"永久熔断"
- Epistemic: **Validated Pattern**（与 webwright 完成授权门/guardian 策略门形成跨项目对照候选，见 Candidates C-02）

## KO-05｜队列化异步平台架构（主题簇）｜L4
**Cognitive Model**：高吞吐观测平台 = 请求路径与执行路径完全解耦，每条耗时链路独立队列 + 死信重试 + 消费隔离。
- aggregation_rule: R4 主题簇——EK-27/28/29/30/31/32 围绕"worker 队列架构"覆盖装配/隔离/重试/失败跟踪/egress/运维六维度
- 证据：worker/src/app.ts:1-60 / worker/src/queues/*（25+ 队列）/ DeadLetterRetryQueue
- 反例攻击：Secondary 队列语义未在代码注释中明确（主/备是冗余还是分流未知）——模型不承诺 Secondary 的具体故障转移语义（见 Candidates C-04）
- Epistemic: **Validated Pattern**（架构形态可证；部分语义未证）

## KO-06｜供应链信任边界（不变量簇）｜L4
**Cognitive Model**：开源平台的供应链安全 = 时间延迟（新依赖观察期）+ 构建白名单 + 已知漏洞钉死 + 上游缺陷本地打丁，四层叠加构成信任边界。
- aggregation_rule: R3 不变量簇——EK-40/41/42/43 汇聚到"依赖进入平台的最小信任门槛"不变量；EK-44 治理约束开发侧
- 证据：pnpm-workspace.yaml（minimumReleaseAge/allowBuilds/overrides/patchedDependencies 全部带注释）
- 反例攻击：延迟策略有 Exclude 白名单（next/turbo 等）——信任边界对"必要工具链"开口，非绝对
- Epistemic: **Validated Pattern**

## KO-07｜双存储分层模型（主题簇）｜L3
**Pattern**：观测平台用"元数据（关系型）+ 事件（列式）"双存储：Postgres 管组织/密钥/配置/版本，ClickHouse 管 trace/observation 事件流，查询层按需路由。
- aggregation_rule: R4 主题簇——EK-33/34/35/36/37/38/39 覆盖事件存储/元数据/RBAC/认证/prompt/过滤七互补维度
- 证据：prisma schema（35+ model）/ eventsTable.ts / tableColumnsToSqlFilterAndPrefix
- 反例攻击：非所有事件都进 ClickHouse（媒体/S3 溢出路径）——模型是"主流事件流"，存在例外通道
- Epistemic: **Validated Pattern**（跨项目对照：与 deepseek-harness 的事件存储同构候选，见 Candidates C-03）

## KO-08｜可观测性平台方法论｜L5
**Methodology**：构建 LLM/Agent 观测平台的可操作准则——
1. 先定语义契约：外部语义（level/type/属性）显式映射表，未知值保守降级不掩盖
2. 摄取即事件化：规格化事件类型 + 确定性采样 + 时间感知延迟 + 溢出通道
3. 评估规格化：eval = 结构化问题定义 + 输出 schema 校验，而非自由文本 prompt
4. 治理第一类：运行开关与健康阻断分层，手动可绕过开关不可绕过健康门
5. 异步解耦：全部耗时路径队列化 + 死信重试 + 失败按项目可见
6. 供应链四层：延迟观察 / 构建白名单 / CVE 钉死 / 上游打丁
- aggregation_rule: R3 不变量簇（跨 KO-01~07 汇聚"观测平台核心不变量"）
- Epistemic: **Methodology（langfuse-originated · Strongly evidenced · Cross-project validation pending）** —— 显式不升格为 Law

## Cross-project Candidates（不升入 KO）
见 05_candidates/candidates.md（C-01~C-06）
