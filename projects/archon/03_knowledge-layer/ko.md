# 03 Knowledge Layer — Archon（ARCH-2026-10-21-001）

> 严格区分 Fact/Observation/Hypothesis/Pattern/Cognitive Model/Principle；每个 KO 声明 aggregation_rule（R1-R4）与簇内 EK。

## KO-B01 · live 凭据端点必须 fail-closed（R3 不变量簇）——L4 Cognitive Model
- **aggregation_rule**: R3 不变量簇（constraint/causal 边汇聚）
- **簇内 EK**: EK-B08（/internal/git-credential 非 loopback → fatal）+ EK-B02（弱密钥 boot 拒绝）+ EK-B05（401/503 分离）+ EK-B15（超限 webhook 认证前拒）
- **内容**: 分发**活凭据**的端点（live installation tokens / signing secrets / webhook 处理）在任何模糊配置下必须**拒绝启动而非警告**——"a WARN line in startup logs is too easy to scroll past"（index.ts:512）；警告是给可接受风险留的显式 opt-out（ARCHON_ALLOW_INTERNAL_ON_PUBLIC_BIND=1 需明确 acknowledged）
- **解释范围扩大论证**: 不是 Archon 专属——任何服务把可立即兑现的凭据（token/secret）暴露给网络边界时，同等配置风险应 fail-closed；可迁移到 reverse proxy/密钥服务/agent runtime 凭据注入点
- **反例**: solo 安装（无 GitHub App）可合法绑 0.0.0.0（无 live token 可漏）——不变量范围限定"有活凭据可分发的端点"
- **五条件**: 内聚 ✓（同一安全不变量）/ 跨实例 ✓（B02/B05/B08/B15 四处独立机制）/ 解释扩大 ✓ / 可命名 ✓（fail-closed credential endpoints）/ 可回溯 ✓

## KO-B02 · 控制面认证 = opt-in gate + fail-closed 默认（R2 因果链簇）——L4 Cognitive Model
- **aggregation_rule**: R2 因果链簇（causal 边完整链：B01→B02→B03→B04）
- **内容**: 认证是一条完整决策链而非单个开关：**启用条件（OFF default）→ 密钥强度（弱则 boot 失败）→ API 门控（默认全 401）→ 注册姿态（默认 disabled，allowlist 优先）**。任何一环的"宽松默认"都会使整链失效；反向代理身份头（X-Archon-User）只在已知剥离边界内可信
- **证据链**: auth/config.ts:24→31→86→55；api-request-auth.ts（401/503）
- **解释扩大**: "secure-by-default 的认证链"模式：启用以配置完备性为条件、强度以 boot 失败为闸、暴露面以默认拒绝为底
- **五条件**: 内聚 ✓ / 跨实例 ✓（四函数独立可测）/ 解释扩大 ✓ / 可命名 ✓ / 可回溯 ✓

## KO-B03 · 隔离强度是显式选择的分层（R1 机制簇）——L3 Pattern
- **aggregation_rule**: R1 机制簇（in-place/container/worktree 三实现 + 显式边界声明）
- **簇内 EK**: EK-B10 + EK-B11 + EK-B12
- **内容**: 项目把执行隔离做成**显式选择的分层**：默认最弱（in-place 宿主直跑，诚实声明无隔离）→ opt-in 容器（只读 lower + overlay upper + docker exec，destroy 丢弃 Phase C 前变更）→ git worktree（**明示"not an execution or security sandbox"**）；每层边界在注释/文档显式声明，引擎对无 containerExec 能力的 provider fail-fast
- **解释扩大**: 与"默认就上最强隔离"的 Box/MXC 对照——Archon 选择"默认最弱但**边界诚实** + 能力显式声明 + 缺能力即失败"；两种策略的取舍：最强默认牺牲易用性，诚实分层牺牲"默认安全"但防虚假安全感
- **反例**: in-place 是默认——若用户不知情即以为有隔离，则失败；文档明示缓解
- **五条件**: 内聚 ✓ / 跨实例 ✓（三实现）/ 解释扩大 ✓ / 可命名 ✓ / 可回溯 ✓

## KO-B04 · 凭据最小暴露主题簇（R4 主题簇）——L4 Cognitive Model
- **aggregation_rule**: R4 主题簇（凭据全生命周期覆盖互补维度）
- **簇内 EK**: EK-B13（vendor-canonical env-only）+ EK-B14（主机白名单）+ EK-B15（原始字节签名）+ EK-B16（不借 host + 不进 evidence）
- **内容**: 凭据治理的五条不变量：**单一归属**（vendor 键而非 agent 键，一凭据多 agent）、**最小交付面**（env-only，不替换 home）、**发送前验证**（主机白名单+签名原始字节）、**不借用**（无凭据 refuse）、**不留痕**（token 不进 run evidence/cleartext）
- **解释扩大**: "凭据是注入点的配置"（Box 主线）的治理侧完整实现——不仅边界换真，还管归属/交付/发送/借用/留痕五维
- **五条件**: 内聚 ✓ / 跨实例 ✓（五机制独立）/ 解释扩大 ✓ / 可命名 ✓ / 可回溯 ✓

## KO-B05 · 身份 = 对话/凭据/run 的所有权根（R3 不变量簇）——L4 Cognitive Model
- **aggregation_rule**: R3 不变量簇（B06/B07/B16 汇聚到同一所有权不变量）
- **内容**: 身份不是"登录状态"而是**所有权根**：对话（非 owner 404 防枚举）、凭据（per-user refuse 不借）、run（只以给定身份认证）全部以身份为根；身份缺失与后端故障严格区分（401 vs 503），身份后端故障**不能 fallback 到整个安装**（requireConversationVisibility 严格路径）
- **证据**: api-request-auth.ts（authorizeConversation/requireConversationVisibility）+ #3433/#4282
- **解释扩大**: "身份故障降级"是常见漏洞源（fallback 到放宽权限）——此处明确拒绝：ownership 强制下 401/503 都不能降级到 all
- **五条件**: 内聚 ✓ / 跨实例 ✓ / 解释扩大 ✓ / 可命名 ✓ / 可回溯 ✓

## 分层声明
- L1/L2 → 02 EK Graph（20 条，无损底座）
- L3（KO-B03）/ L4（KO-B01/B02/B04/B05）——无 L5（无方法论级新准则，已有 dsh/box 覆盖；不硬凑）
