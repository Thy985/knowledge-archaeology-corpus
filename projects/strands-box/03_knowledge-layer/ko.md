# 03 Knowledge Layer — strands-box（ARCH-2026-10-23-001）

> 严格区分 Fact/Observation/Hypothesis/Pattern/Cognitive Model/Principle；每个 KO 声明 aggregation_rule。

## KO-B01 · 执行面与决策面分离，默认拒绝由类型与契约共同保证（R2 因果链簇）——L4 Cognitive Model
- **aggregation_rule**: R2 因果链簇（B01→B02→B06→B08→B09→B13 沿 causal/mechanism 边形成完整判定链）
- **簇内 EK**: EK-B01（三拦截点=一个 authority）+ EK-B02（fail-closed 四语义）+ EK-B05（ApprovedPath 类型化）+ EK-B06（统一事件）+ EK-B13（capability fold）
- **内容**: agent 请求经 PEP（进程外三适配器）→ 统一 Request → 唯一 PDP（Dogwood）→ 默认拒绝（permit 必须显式）→ 回执（403+rule id）或执行 → DecisionObserver（非阻塞）→ telemetry；**类型系统（ApprovedPath 编译期防 raw path）与引擎契约（cedar pin 安全契约）共同保证"拒绝不可绕过"**
- **解释范围扩大**: "决策与执行分离 + 类型化请求 + 默认拒绝"是零信任授权面的通用范式（同族：SPIFFE/OPA 的 PDP-PEP 分离、TLS CA 验证）——本仓库以 Rust 类型 + 精确 pin 落实，可迁移到任何 agent 治理运行时
- **反例**: MCP gate 2 精化**非**默认拒绝（B07）——粗授权宽泛是设计决策，说明"默认拒绝"有明确范围边界（粗 gate 拒绝、精化不拒绝）
- **五条件**: 内聚 ✓ / 跨实例 ✓（三适配器独立可测）/ 解释扩大 ✓ / 可命名 ✓ / 可回溯 ✓

## KO-B02 · 机密性边界 = 边界持有，不随授权扩散（R3 不变量簇）——L4 Cognitive Model
- **aggregation_rule**: R3 不变量簇（B10/B11/B12 经 causal/constraint 边汇聚到"agent 永不持密"不变量）
- **簇内 EK**: EK-B10（Vault 边界注入 5 模式）+ EK-B11（box 是凭证边界，无 per-process selection）+ EK-B12（FORBIDDEN_PATHS + validate_grant）
- **内容**: 真实密钥在**网络边界**解析并注入出站请求（5 种模式，sign_request 于 source 层）；agent 环境只见 placeholder；凭据路径（~/.aws/~/.ssh/keychain/browser profile）经 FORBIDDEN_PATHS **永不授予**（仅 operator 精确命名单点例外）；不按进程选择凭证（gateway/broker 无法区分调用者——"A process that must not get a credential goes in another box"）
- **解释范围扩大**: "授权可以扩散，机密不扩散"——权限允许 agent 触达资源，但触达凭据的**能力**被物理边界隔离；这是对"allowlist 即安全"的反驳（tenet 2："An allowlisted endpoint is still an exfiltration channel if anything in the sandbox can use it"）；与 aegis 的 assessed fields（评估独立于声明）同构：信任判据不来自被信任方
- **反例**: 无 per-process selection 意味着同一 box 内所有工具共享凭据路由（粗粒度是设计取舍——需更多 box 细分）
- **五条件**: 内聚 ✓ / 跨实例 ✓ / 解释扩大 ✓ / 可命名 ✓ / 可回溯 ✓

## KO-B03 · fail-closed 是逐层机制而非顶层口号（R1 机制簇）——L4 Cognitive Model
- **aggregation_rule**: R1 机制簇（B02/B03/B04/B05/B13 共享"每个可失败点显式闭合"机制，跨 policy/egress/config 三子系统）
- **簇内 EK**: EK-B02（四拒绝语义）+ EK-B03（引擎安全 pin）+ EK-B04（SELF_DEFENDED + refuse_removed_keys）+ EK-B05（ApprovedPath）+ EK-B13（MutationCollision）
- **内容**: fail-closed 的落实 = 策略包未装（PolicyPending）拒绝 + 引擎版本漂移（pin）拒绝 + 策略文件自身保护 + 旧配置键按名拒绝 + 路径类型化 + 网关跨能力冲突拒绝——**每个可能让"本应失败的东西放行"的点都有显式闭合**；测试（fail_closed_guards）专门防 fail-open 类回归（"a guard that lets one of these through has reopened the fail-open class"）
- **解释范围扩大**: "fail-closed"是可操作工程属性：不是声明默认拒绝，而是枚举全部可失败点并逐一验证闭合路径（同族：TLS 证书验证的每个失败分支、Kubernetes admission webhook 的 failurePolicy=Fail）；对照 agent-aegis 的组件失败 warning 降级（G16）——**两种哲学：治理组件可选择性失败（aegis）vs 机制失败即拒绝（Box）**
- **反例**: MCP 双门 gate2 非默认拒绝 + egress Resolve 的 mcp_hosts PolicyPending 放行（MCP 未装策略时白名单主机可连）——存在**白名单例外面**（B07/B13 文档化）
- **五条件**: 内聚 ✓ / 跨实例 ✓（三个子系统独立闭合）/ 解释扩大 ✓ / 可命名 ✓ / 可回溯 ✓

## KO-B04 · 接口冻结 + 测试钉行为 = 可演进的 fail-closed 治理模式（R4 主题簇）——L3 Pattern
- **aggregation_rule**: R4 主题簇（B16/B18/B19/B20 围绕"治理机制"覆盖互补维度）
- **簇内 EK**: EK-B16（interface-frozen + ADR 降格）+ EK-B18（reload 调用方认证）+ EK-B19（黑盒套件 + fail-open 回归钉死）+ EK-B20（依赖面诚实声明）
- **内容**: 治理三层：**接口冻结**（两类变更需用户批准，agent 不得自批，feature-critic BLOCKER 级）→ **ADR 降格**（decisions.md 自称 historical record，"code is the only authority"）→ **测试钉死**（fail_closed_guards 使 fail-open 回归成为编译/测试失败）；reload 授权明确由调用方承担
- **解释范围扩大**: 对"agent 系统如何治理自己"的可复用答案：可演进性来自冻结面 + 文档权威降格 + 行为测试三层并存（同族：Tafcm AGENTS.md 权限矩阵、OpenClaw 治理）；与 aegis 的 AGEF/AGP 自述标准对照：**Box 选择文档降格 + 冻结面，aegis 选择自述标准 + conformance 自测**——两种治理契约形态
- **五条件**: 内聚 ✓ / 跨实例 ✓ / 解释扩大 ✓ / 可命名 ✓ / 可回溯 ✓

## KO-B05 · 文档权威性降格是诚实工程标记（R3 不变量簇）——L4 Cognitive Model（cross-project）
- **aggregation_rule**: R3 不变量簇（B16 的 ADR 降格 + decisions.md 自述 + 测试优先证据，汇聚到"文档≠规范"不变量）
- **簇内 EK**: EK-B16（decisions.md "The code is the only authority"）+ EK-B19（测试是行为权威）+ 跨项目证据（aegis AGEF/AGP Draft 自述 + conformance 自测）
- **内容**: 项目文档明确声明自身不是规范（"Read this as a historical record, not as a specification…the code is the only authority"）——文档权威性**主动降格** + 行为以测试为准，是工程诚实性标记；与 aegis 自述标准（AGEF/AGP Draft + 自我 conformance）同构但形态相反：Box 降格文档、aegis 升格自述标准
- **解释范围扩大**: 跨项目（aegis 10-22 + Box 10-23 两仓库独立证据）：**任何项目内"标准/规范文本"必须与其实现证据分层标注**——规范文本（Draft/historical）、实现一致性（conformance/测试）、生态采纳（第三方）三者的证据强度不同；Hypothesis 不冒充 Principle
- **反例**: decisions.md 自己警告条目可能落后（"an entry can fall behind the code"）——降格声明本身是可证伪的（若后续条目被当规范引用则违背）
- **五条件**: 内聚 ✓ / 跨实例 ✓（两仓库独立证据）/ 解释扩大 ✓ / 可命名 ✓ / 可回溯 ✓

## 分层声明
- L1/L2 → 02 EK Graph（20 条，无损底座）
- L3（KO-B04）/ L4（KO-B01/B02/B03/B05）——无 L5（方法论级无新准则，不硬凑）
