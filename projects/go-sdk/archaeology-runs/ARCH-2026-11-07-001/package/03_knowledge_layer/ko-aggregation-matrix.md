# 03.3 KO 聚合规则矩阵

## 1. 总表

| KO | 标题 | Level | 聚合规则 | 簇内 EK | 边类型 | 簇规模 |
|---|---|---|---|---|---|---|
| KO-01 | 协议迁移：探测优先/无损回退 | L3 Pattern | R2 因果链 | 01,02,03,04,05 | causal + dependency + mechanism + contrast | 5 |
| KO-02 | 无状态化 = 请求级会话 | L4 Model | R4 主题簇 | 05,06,07,09,10,17 | mechanism + contrast + dependency | 6 |
| KO-03 | 会话身份绑定是传输层不变量 | L3 Pattern | R3 不变量簇 | 08,09,26,27 | subsystem + dependency + constraint | 4 |
| KO-04 | 版本是三平面事实 | L3 Pattern | R2 因果链 | 17,18,03,04 | mechanism + constraint | 4 |
| KO-05 | 门禁有效性取决于执行点 | L4 Model | R3 不变量簇 | 12,13,14,15 | contrast + causal + mechanism | 4 |
| KO-06 | 重放依赖"不得部分重放"契约 | L3 Pattern | R2 因果链 | 19,20,21 | subsystem + dependency | 3 |
| KO-07 | 交互折叠为多轮结果 | L3 Pattern | R1 机制簇 | 14,15,16 | mechanism | 3 |
| KO-08 | 缺口显性化是治理资产 | L3 Pattern | R4 主题簇 | 28,29,30 | mechanism + subsystem | 3 |

## 2. 簇成立五条件核对

| KO | 内聚性 | 跨实例性（≥2 独立文件/子系统） | 解释范围扩大 | 可命名 | 可回溯 EK→Evidence |
|---|---|---|---|---|---|
| KO-01 | ✅ 同一"迁移"问题 | ✅ client.go / server.go / shared.go | ✅ 连接→发现→协商→兼容 | ✅ 探测-回退 | ✅ E-22/23/26/28 |
| KO-02 | ✅ 同一"无状态"语义 | ✅ streamable.go / client.go / server.go | ✅ 传输/状态/身份/配置 | ✅ 请求级会话 | ✅ E-09/13/19/46 |
| KO-03 | ✅ 同一安全不变量 | ✅ streamable.go / auth.go | ✅ 会话层/授权层/无状态边界 | ✅ 会话身份绑定 | ✅ E-15/48 |
| KO-04 | ✅ 同一"版本事实" | ✅ streamable.go / client.go / server.go / shared.go | ✅ header/body/握手 | ✅ 三平面 | ✅ E-45/46/28 |
| KO-05 | ✅ 同一"门禁"问题 | ✅ server.go（检查/通知） / protocol.go（MRTR） | ✅ 旧协议门禁→新协议交互约束 | ✅ 执行点门禁 | ✅ E-30/31/32 |
| KO-06 | ✅ 同一"重放"契约 | ✅ event.go / streamable.go | ✅ 存储/并发/清理 | ✅ 不得部分重放 | ✅ E-35/38/39 |
| KO-07 | ✅ 同一"交互折叠"机制 | ✅ server.go / protocol.go | ✅ 禁令→结果类型→过载 | ✅ 交互折叠 | ✅ E-32/33/34 |
| KO-08 | ✅ 同一"负向清单"主题 | ✅ conformance/ / docs/ / LICENSE | ✅ 测试/API/合规 | ✅ 缺口显性化 | ✅ E-49/50/51 |

## 3. 防退化检查

- ❌ **无"同子系统=聚合理由"**：每个 KO 的聚合理由都是机制/因果/不变量/主题，而不是"都属于 streamable"。
- ❌ **无拍脑袋分组**：8 个 KO 全部有 `aggregation_rule` 与簇内边。
- ✅ **无孤立 EK**：30/30 有至少一条边。
- ✅ **KO 均可回溯**：每个 KO 的 Evidence 列指向具体 E-xx。
- ⚠️ **KO-05 的 Epistemic 不是纯 Fact**：`checkInitialized` 恒 nil 是 Fact；"为什么注释"是注释里的自述，无法独立验证，因此整体标 `Observation`，不写成"作者为测试便利牺牲安全"的动机判断。

## 4. 升维审计（Abstraction Promotion Gate 摘要）

| KO | 从 L1/L2 升到 | 是否解释范围扩大 | 是否过度升维 | 判定 |
|---|---|---|---|---|
| KO-01 | L2 → L3 | 是（覆盖三文件） | 否 | 保留 L3 |
| KO-02 | L2 → L4 | 是（覆盖四层） | 否，但跨项目未验 | L4 + pending |
| KO-03 | L2 → L3 | 是（会话+授权） | 否 | 保留 L3 |
| KO-04 | L2 → L3 | 是（三平面） | 否 | 保留 L3 |
| KO-05 | L2 → L4 | 是（两代协议） | 边界：单项目反例 | L4 + pending，不写"普遍定律" |
| KO-06 | L2 → L3 | 是（存储+并发+清理） | 否 | 保留 L3 |
| KO-07 | L2 → L3 | 是（协议+结果类型+过载） | 否 | 保留 L3 |
| KO-08 | L2 → L3 | 是（测试+API+合规） | 否 | 保留 L3 |
