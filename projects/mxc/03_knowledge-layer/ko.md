# 03 · Knowledge Layer — MXC KO（KO-M01~M05）

> 每个 KO 声明 aggregation_rule（R1-R4）+ 簇内 EK + 簇成立论证。全部 Cross-project validation pending（单项目证据，不冒充已验证 Principle）。

## KO 聚合规则矩阵

| KO | 层 | aggregation_rule | 簇内 EK | 边类型 | 解释范围扩大论证 |
|---|---|---|---|---|---|
| KO-M01 | L4 | **R1 机制簇**（身份/可审计性机制跨独立子系统出现） | M01/M02/M07 | mechanism | "每次运行的身份可独立审计"是跨平台一致属性：Linux Bubblewrap 与 Windows ProcessContainer 都发 PolicyHash + 审计 |
| KO-M02 | L4 | **R2 因果链簇**（策略→校验→执行→审计闭环） | M01→M19→M06→M02 | causal | 完整策略生命周期故事：身份锚定→校验 gate→执行→审计落盘 |
| KO-M03 | L4 | **R1 机制簇**（"配置不能自我授权"跨实验性/遥测出现） | M08/M13 | mechanism | 授权与限制分离原则在实验性后端与遥测两处独立实现 |
| KO-M04 | L3 | **R3 不变量簇**（文件系统策略以路径规范化为不变量） | M15/M16 | constraint | 所有后端共享"路径必须规范化"的安全不变量 |
| KO-M05 | L3 | **R4 主题簇**（契约治理互补维度） | M17/M18/M20 | subsystem | 版本化/精确窗口/非 SDK 消费覆盖"契约如何发布与消费"整体 |

## KO 明细

### KO-M01 · 运行身份可审计：策略身份与审计记录解耦于执行（L4, R1）
- **一句话**: 每次沙箱运行都产生独立于执行结果的规范化策略身份（PolicyHash）与结构化审计记录——身份不是执行副作用，而是策略的指纹
- **簇内**: EK-M01（PolicyHash allow-list 投影 + 穷举）+ EK-M02（闭合枚举审计）+ EK-M07（sink 隔离）
- **反例攻击**: ①"哈希无密钥=预言机"→项目明示接受但排除全部敏感输入（M01 文档 §Residual disclosure）；②"审计会被篡改"→本地诊断记录无防篡改承诺（边界在 audit.rs 文档，未超卖）
- **Cross-project pending**: 与 RAMPART 的"终局证据分离"（KO-R01）同族——两个项目都把"证明发生了什么"与"发生了什么"解耦，但 MXC 用哈希指纹、RAMPART 用不可变版本化 schema。跨项目验证待 RAMPART/MXC/AGT 三方对照

### KO-M02 · 策略生命周期闭环：身份→校验→执行→审计（L4, R2）
- **一句话**: 策略从声明到执行再到留痕构成因果闭环，每一环都有显式 gate——校验拒绝不静默、审计记录不达调用方输出
- **簇内**: EK-M01→EK-M19（校验分层）→EK-M06（执行双路径）→EK-M02（审计）
- **反例攻击**: ①"dry_run 校验路径跳过"→streaming 显式拒绝 dry_run；②"后端不支持网络规则被忽略"→validator 显式 ScriptResponse::rejected
- **Cross-project pending**: 与 agent-governance-toolkit 的"无状态策略引擎拦截每个动作"对照——MXC 是 OS 级、AGT 是应用层，机制不同但闭环形状相同

### KO-M03 · 授权与限制分离：配置不能自我授权（L4, R1）
- **一句话**: 扩展攻击面（实验性后端）与遥测采集都要求调用方显式 opt-in，策略文档本身永远不能授予自己这些权限
- **簇内**: EK-M08（experimental 是参数非 JSON 字段）+ EK-M13（telemetry 非策略字段）
- **反例攻击**: "有人把 experimental: true 写进 JSON 策略"→schema 无该字段（spawn_one_shot_json 的 experimental 是函数参数）；"遥测随配置开关"→policy.rs 明示 telemetry 不是策略字段
- **Cross-project pending**: 与 dsh 的 approval seam、silver-shield 权限模型同族——"权限来源必须在被授权者之外"

### KO-M04 · 路径规范化是沙箱文件系统策略的不变量（L3, R3）
- **一句话**: 无论后端用挂载（Bubblewrap）还是 DACL（AppContainer），策略执行前路径必须被折叠到唯一规范形态，否则别名（symlink/junction/8.3）成为逃逸通道
- **簇内**: EK-M15（canonicalize_path 全别名折叠）+ EK-M16（DACL tier 恢复）
- **反例攻击**: "dangling symlink"→分类为 Unknown（可解析但不可解析=不清不楚拒绝）；"未创建叶"→canonicalize_allowing_absent_tail 解析最深存在祖先
- **Cross-project pending**: 与 E2B/云沙箱、用户 E2E-CLI 路径处理对照

### KO-M05 · 契约 = 不可变发布物，消费端精确匹配（L3, R4）
- **一句话**: 沙箱配置契约像 API 一样版本化：发布即冻结、消费端精确匹配版本、实验版与发布版分离——schema 是权威源而非文档
- **簇内**: EK-M17（版本精确匹配）+ EK-M18（min/maxSupported 窗口）+ EK-M20（独立执行器走同一契约）
- **反例攻击**: "schema 可以就地改"→文件自述 PUBLISHED IMMUTABLE；"语义版本比较"→parse_exact 拒绝
- **Cross-project pending**: 与 RAMPART 的 trace-compatibility 决策门、APT 生态的 schema 版本化同族

## 层纪律
- 全部 KO 标 `Cross-project validation pending`（单项目证据，无跨项目验证不升 Law）
- L4×3（M01/M02/M03）+ L3×2（M04/M05）；无 L5（方法论需跨项目验证，未达）
- 每个 KO 簇内 EK 可回溯（见矩阵）
