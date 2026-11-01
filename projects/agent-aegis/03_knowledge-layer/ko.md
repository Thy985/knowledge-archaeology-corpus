# 03 Knowledge Layer — agent-aegis（ARCH-2026-10-22-001）

> 严格区分 Fact/Observation/Hypothesis/Pattern/Cognitive Model/Principle；每个 KO 声明 aggregation_rule。

## KO-G01 · 治理判定 = 声明 × 独立评估 × 单调信任链（R2 因果链簇）——L4 Cognitive Model
- **aggregation_rule**: R2 因果链簇（G03→G07→G08 沿 causal/mechanism 边形成完整判定链）
- **簇内 EK**: EK-G03（声明 vs 评估分离）+ EK-G07（单调约束）+ EK-G08（四步评估+复合解释）+ EK-G12（能力矩阵）
- **内容**: 一次治理判定 = agent 自述（untrusted declared fields）→ 独立评估（assessed fields，agent 无权 author）→ 委托链单调校验 → 复合决策；**justification_gap（自述影响 vs 独立计算影响的距离）是审批核心判据**，congruence<0.5 提示"声明参数矛盾"
- **解释范围扩大论证**: 不只是 policy 匹配——这是"代理声明-系统验证"认知范式在 agent 权限的落地（同族：TLS 证书自述 vs CA 验证、SBOM 声明 vs 扫描验证）；可迁移到任何"由不可信方声明、由系统独立核验"的授权面
- **反例**: 无 assessor 时（`_assess=False`）claim 未经独立评估即进 policy——诚实降级路径（初始化依赖配置）
- **五条件**: 内聚 ✓ / 跨实例 ✓（三模块独立可测）/ 解释扩大 ✓ / 可命名 ✓ / 可回溯 ✓

## KO-G02 · 策略即数据 + 防遮蔽缓存（R1 机制簇）——L3 Pattern
- **aggregation_rule**: R1 机制簇（G04/G05 共享"策略数据化"机制，跨 policy.py/policy.example.yaml 两实例）
- **簇内 EK**: EK-G04（LRU+防遮蔽）+ EK-G05（YAML+merge）+ EK-G06（LLM 生成）+ EK-G10（fail-safe 默认）
- **内容**: 策略以数据形态存在（YAML 规则集，可 merge 覆盖），运行时编译为可缓存判定；**缓存必须对"含条件规则的 action"禁用**（防无条件命中遮蔽条件规则）；无匹配落 `<default>`（fail-safe 由 AGP conformance 钉住）
- **解释扩大**: 与 agent-governance-toolkit（无状态策略引擎）、OPA/Cedar 同族——"策略与执行分离 + 缓存正确性依赖规则结构"是可复用工程约束；LLM 生成策略（G06）是该模式的新变体（生成物必须过 _validate_policy_dict）
- **反例**: merge 的 defaults 取 base——若环境覆盖想改默认风险级别则不支持（语义边界）
- **五条件**: 内聚 ✓ / 跨实例 ✓ / 解释扩大 ✓ / 可命名 ✓ / 可回溯 ✓

## KO-G03 · 零信任会话验证 + 可逆插桩（R4 主题簇）——L4 Cognitive Model
- **aggregation_rule**: R4 主题簇（G01/G12/G15/G18 围绕"接入与信任面"覆盖互补维度）
- **簇内 EK**: EK-G01（惰性零成本）+ EK-G02（三梯度接入）+ EK-G12（能力矩阵）+ EK-G15（每请求验证）+ EK-G18（可逆补丁）
- **内容**: 治理接入设计为**渐进梯度**（env 一行 → auto_instrument → 显式生命周期）+ **可逆**（_deactivate 撤销补丁）；信任模型为**每请求验证**（ZeroTrustAgent）+ **能力声明即权限根**（capabilities frozenset）
- **解释范围扩大**: "治理必须低摩擦接入且可装卸"——与 Archon 的 opt-in gate（10-21）对照：aegis 用插桩实现低摩擦，Archon 用配置 gate 实现 fail-closed；两种接入哲学（默认开-低摩擦 vs 默认关-显式开）互补而非互斥
- **五条件**: 内聚 ✓ / 跨实例 ✓ / 解释扩大 ✓ / 可命名 ✓ / 可回溯 ✓

## KO-G04 · MCP 攻击面是独立安全域（R1 机制簇）——L3 Pattern
- **aggregation_rule**: R1 机制簇（G13/G14 共享"MCP 攻击面检测"机制，跨 shadow/escalation 两实现）
- **簇内 EK**: EK-G13（shadow 4 类）+ EK-G14（升级链滑动窗口）+ EK-G10（fail-safe 落点）
- **内容**: MCP 多 server 拓扑暴露**独立攻击面**：同名工具影子攻击（exact/typosquat/description_override/capability_claim 四向量）与跨 server 升级链（source→sink）需要专门检测器；检测器产出 findings 进入决策（G08 selection audit）
- **解释扩大**: MCP 生态（工具互连）的信任边界 = 工具名空间 + 调用序列——与本 corpus mcp 卡、MCP 信任边界卡连接；这是"协议层安全域"模式（同族：DNS 投毒检测、供应链 typosquat 检测）
- **反例**: shadow 检测依赖 trusted_servers 白名单配置——未配置时能力降级（需验证默认值）
- **五条件**: 内聚 ✓ / 跨实例 ✓（两个独立检测器）/ 解释扩大 ✓ / 可命名 ✓ / 可回溯 ✓

## KO-G05 · 自述标准必须与实现分开标注（R3 不变量簇）——L4 Cognitive Model
- **aggregation_rule**: R3 不变量簇（G17 + conformance 测试汇聚到"标准真实性"不变量）
- **簇内 EK**: EK-G17（AGEF/AGP 自述 Draft）+ EK-G16（组件降级）+ conformance 测试证据
- **内容**: AGEF/AGP 是**仓库内自述标准**（Status: Draft, 2026-03-24），conformance 测试以**本引擎自身**验证——"自我一致"不等于"生态采纳"；厂商自述（"AI governance 的 SARIF/CEF"）无第三方证据，须标 Hypothesis 而非 Principle
- **解释范围扩大**: 任何项目内"标准"必须分三个认知层：规范文本（Draft）、实现一致性（conformance 自测）、生态采纳（第三方）；三者的证据强度完全不同——此不变量可迁移到任何自造标准项目
- **反例**: README 徽章（OpenSSF Best Practices）是外部程序——但那是项目治理参与，不是 AGEF/AGP 生态采纳
- **五条件**: 内聚 ✓ / 跨实例 ✓ / 解释扩大 ✓ / 可命名 ✓ / 可回溯 ✓

## 分层声明
- L1/L2 → 02 EK Graph（18 条，无损底座）
- L3（KO-G02/G04）/ L4（KO-G01/G03/G05）——无 L5（方法论级无新准则，不硬凑）
