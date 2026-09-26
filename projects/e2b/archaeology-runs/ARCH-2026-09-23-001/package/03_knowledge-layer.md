# 03 Knowledge Layer — E2B（Generalized KO，窄尖顶）

> v3.1 契约：每个 KO 是 EK 图上的簇，必须声明 `aggregation_rule`（R1-R4）。簇成立五条件：内聚性/跨实例性/解释范围扩大/可命名/可回溯。
> L3/L4 升维均经 Abstraction Promotion 论证（06 层有独立判定记录）。

---

## KO-01 平台秘密不落地：占位符替代 + write-only + 签名寻址
- **knowledge_layer**: generalized · **abstraction**: L4 · **value**: A
- **claim**：在"代码执行沙箱 + 外部服务访问"系统中，秘密（API 令牌/工作量身份/服务账号）以**占位符 + 平台侧按请求签发**的方式流动，SDK 永不持有秘密值；外部可寻址资源用签名 URL 而非秘密查询参数。
- **epistemic_status**: Cognitive Model（本项目强证据；跨项目验证 pending，见 Candidates C-06）
- **aggregation_rule**: {rule: R3 不变量簇, cluster_eks: [EK-05, EK-13, EK-15, EK-17, EK-18], edges: [subsystem/mechanism: 令牌链 EK-05↔EK-13↔EK-15↔EK-18；dependency: EK-17→EK-05], naming: "全部成员汇聚到同一不变量：秘密值不出平台", scope_expansion: "单条 EK 只描述一级令牌；簇揭示了跨 API key/envd/IAM/Secret/签名 URL/MCP 六处统一的秘密处置策略"}
- **derivation.facts**: [EK-05, EK-13, EK-15, EK-17, EK-18]
- **scope**: applies_when: 沙箱/远端执行平台向用户代码暴露外部凭据时；does_not_apply_when: 纯本地单进程应用
- **confidence**: high（同机制在 6 个独立文件出现）

## KO-02 生命周期契约前置校验：歧义请求不进服务端
- **knowledge_layer**: generalized · **abstraction**: L4 · **value**: A
- **claim**：托管资源（沙箱）的生命周期选项（超时动作/快照语义/自动恢复）在客户端以判别联合 + 交叉约束完成语义校验，非法组合在本地抛错，绝不把"服务器会怎么解释"留给远端——客户端契约前置。
- **epistemic_status**: Validated Pattern
- **aggregation_rule**: {rule: R2 因果链簇, cluster_eks: [EK-01, EK-02, EK-03], edges: [causal: EK-01(API 面)→EK-02(强校验)→EK-03(版本门)], naming: "完整链：入口→校验→运行时版本确认", scope_expansion: "成员 EK 各自描述一段；簇揭示'客户端先裁决、服务端后确认'的完整生命周期契约"}
- **derivation.facts**: [EK-01, EK-02, EK-03]
- **scope**: applies_when: 管理远端有状态资源的 SDK；does_not_apply_when: 纯查询类 API
- **confidence**: high

## KO-03 保守降级错误归因：探测后再断言失败原因
- **knowledge_layer**: generalized · **abstraction**: L4 · **value**: A
- **claim**：当远程调用以"连接中断"这一模糊信号失败时，系统不直接断言原因，而是发起**独立健康探测**：确认目标确实消失才归因"资源已死"，探测失败则保守抛原错误——失败归因与失败本体分离。
- **epistemic_status**: Validated Pattern（三处独立实例 + 测试证实）
- **aggregation_rule**: {rule: R1 机制簇, cluster_eks: [EK-07, EK-19, EK-20, EK-21, EK-22, EK-28], edges: [mechanism: 归因机制跨 envd REST/gRPC/commands/code-interpreter 四实例], naming: "同一'探测→归因'机制在 4 个独立通道出现", scope_expansion: "单条 EK 是一个通道的归因；簇揭示这是 SDK 层的统一失败哲学"}
- **derivation.facts**: [EK-07, EK-19, EK-20, EK-21, EK-22, EK-28]
- **scope**: applies_when: 长连接/流式 API 需要区分'远端没了'与'网络抖了'；does_not_apply_when: 单请求 REST（错误码已自解释）
- **confidence**: high（killedSandbox.test.ts 直接验证）

## KO-04 两阶段超时 + 有界并发：连接治理骨架
- **knowledge_layer**: generalized · **abstraction**: L3 · **value**: B
- **claim**: 流式远程调用用"握手超时（启动）→ 空闲读超时（只限网络侧）"两阶段控制；配合 429 限流重试（可重放才重试）与 FIFO 信号量（0=可关闭），构成 SDK 的确定性连接治理骨架。
- **epistemic_status**: Validated Pattern
- **aggregation_rule**: {rule: R4 主题簇, cluster_eks: [EK-08, EK-09, EK-10, EK-26], edges: [dependency: EK-09→EK-10; mechanism: EK-08↔EK-26 两阶段超时], naming: "连接生命周期治理的互补四维度：超时/重试/并发/执行预算", scope_expansion: "成员 EK 各管一个维度；簇给出完整治理模型"}
- **derivation.facts**: [EK-08, EK-09, EK-10, EK-26]
- **scope**: applies_when: SDK/客户端需要确定性资源上限；does_not_apply_when: 内网低延迟调用
- **confidence**: medium（inflight slot 提前释放 TODO 未解决，见 C-02）

## KO-05 沙箱网络面：过滤先行 + fail-closed 隧道 + 声明式安全边界
- **knowledge_layer**: generalized · **abstraction**: L4 · **value**: A
- **claim**：沙箱出网控制遵循"允许/拒绝过滤 → 每域变换 → 可选隧道"的管道序；隧道在宿主层执行（沙箱内不可见、不可绕开），不可达即 fail-closed；安全边界（控制面操作不得经公共 URL 到达）由规范声明标记自动生成拒绝列表。
- **epistemic_status**: Cognitive Model
- **aggregation_rule**: {rule: R4 主题簇, cluster_eks: [EK-11, EK-12, EK-24, EK-30], edges: [causal: EK-11→EK-12；mechanism: EK-12↔EK-30 安全边界声明化], naming: "沙箱网络从配置模型、隧道执行到边界声明的完整栈", scope_expansion: "成员 EK 分别是网络三块/隧道/寻址/边界；簇给出'出网=有序管道'整体模型"}
- **derivation.facts**: [EK-11, EK-12, EK-24, EK-30]
- **scope**: applies_when: 多租户代码执行平台的网络隔离；does_not_apply_when: 无外部网络暴露的批处理
- **confidence**: high

## KO-06 防御性输入卫生：调用方输入不可信
- **knowledge_layer**: generalized · **abstraction**: L3 · **value**: B
- **claim**: 对外部/半可信输入（用户对象、解析 JSON、env、token 名）做系统性卫生处理：Proxy 守卫运行时探测属性、defineProperty 防 `__proto__` 原型污染、字符集校验防占位符注入、env 非法值大声失败——同一"输入不可信"哲学跨四个独立子系统。
- **epistemic_status**: Validated Pattern
- **aggregation_rule**: {rule: R1 机制簇, cluster_eks: [EK-14, EK-16, EK-23], edges: [mechanism: 输入卫生机制跨 iam.ts/secret.ts/mergeOpts/metadata.ts], naming: "同一防护哲学的多实例", scope_expansion: "成员各防一种污染向量；簇揭示统一输入卫生策略"}
- **derivation.facts**: [EK-14, EK-16, EK-23]
- **scope**: applies_when: SDK/框架处理用户回调与半可信输入；does_not_apply_when: 全可信内部代码
- **confidence**: high

## KO-07 规范所有权与生成契约：仓库是规范消费者
- **knowledge_layer**: generalized · **abstraction**: L3 · **value**: B
- **claim**: 多语言 SDK 仓库把 API 规范放在独立上游仓库，以 pin commit + 代码生成器同步，仓库内禁止手改规范；SDK 变更强制跨语言等效——规范所有权边界与双实现契约共同防止"客户端与服务器契约漂移"。
- **epistemic_status**: Validated Pattern
- **aggregation_rule**: {rule: R4 主题簇, cluster_eks: [EK-29, EK-31], edges: [dependency: EK-31→EK-29], naming: "规范同步 + 双实现同步 = 仓库治理两个不变量", scope_expansion: "成员 EK 各管一个同步方向；簇揭示'契约漂移'防御整体设计"}
- **derivation.facts**: [EK-29, EK-31]
- **scope**: applies_when: 跨语言 SDK + 独立服务端仓库；does_not_apply_when: 单语言单体仓库
- **confidence**: high

## KO-08 执行会话可恢复性：状态保留 + 自愈 + 显式重连
- **knowledge_layer**: generalized · **abstraction**: L3 · **value**: B
- **claim**: 代码执行沙箱把"会话状态"与"执行进程"解耦：上下文（Jupyter context）保留变量状态跨调用；内核被 kill 后由 systemd 自愈恢复；会话可经 connect 显式重建——测试把这三条可恢复性承诺固化为可执行断言。
- **epistemic_status**: Validated Pattern（测试证实）
- **aggregation_rule**: {rule: R4 主题簇, cluster_eks: [EK-27, EK-32], edges: [causal: EK-27→EK-32（实现→测试证实）], naming: "可恢复性三承诺：状态保留/自愈/重连", scope_expansion: "成员 EK 分别是实现与测试；簇给出完整可恢复性契约"}
- **derivation.facts**: [EK-27, EK-32]
- **scope**: applies_when: 长会话代码执行/远程内核；does_not_apply_when: 一次性无状态调用
- **confidence**: high（systemd.test.ts 直接验证 kill 后自愈）

## KO-09 SDK 连接架构：双通道 + 多 client + 本地调试形态
- **knowledge_layer**: generalized · **abstraction**: L3 · **value**: B
- **claim**: SDK 以"同一资源对象、多连接形态"设计：REST + gRPC 双通道、boundOpts 静态绑定支持同进程多 client 共存、debug 模式本地短路——连接形态与资源语义解耦。
- **epistemic_status**: Observation（架构事实归纳，未跨项目验证）
- **aggregation_rule**: {rule: R4 主题簇, cluster_eks: [EK-04, EK-06, EK-25], edges: [subsystem: EK-06↔EK-04（都是连接形态）; mechanism: EK-25↔EK-01], naming: "连接形态三视图：通道/隔离/调试", scope_expansion: "成员 EK 各是一种连接形态；簇给出 SDK 连接架构整体"}
- **derivation.facts**: [EK-04, EK-06, EK-25]
- **scope**: applies_when: 设计多形态客户端 SDK；does_not_apply_when: 单形态内部服务
- **confidence**: medium

---

## 聚合矩阵（KO ← EK）

| KO | 规则 | 簇内 EK | 簇规模 |
|---|---|---|---|
| KO-01 | R3 | EK-05,13,15,17,18 | 5 |
| KO-02 | R2 | EK-01,02,03 | 3 |
| KO-03 | R1 | EK-07,19,20,21,22,28 | 6 |
| KO-04 | R4 | EK-08,09,10,26 | 4 |
| KO-05 | R4 | EK-11,12,24,30 | 4 |
| KO-06 | R1 | EK-14,16,23 | 3 |
| KO-07 | R4 | EK-29,31 | 2 |
| KO-08 | R4 | EK-27,32 | 2 |
| KO-09 | R4 | EK-04,06,25 | 3 |

- EK 覆盖：32/32（无游离 EK；无 EK 重复归属）
- 平均簇规模：3.56（健康区间 3~12；KO-07/KO-08 规模 2 属小簇，理由见各 KO scope_expansion）
- 聚合规则覆盖率：100%
