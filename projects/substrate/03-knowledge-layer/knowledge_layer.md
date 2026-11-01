# Knowledge Layer — Agent Substrate（03 KO）

> 7 条 Core KO，每条声明 aggregation_rule（R1-R4）+ 簇内 EK + 解释范围扩大论证 + epistemic 状态。
> 铁律：Hypothesis 不冒充 Fact；单项目证据 → `Cross-project validation pending`。

## KO-S01 快照-恢复生命周期三件套（分级快照 × 幂等 checkpoint × 本地/远端双持久化）
- aggregation_rule: **R1 机制簇** — EK-S02（分级快照）、EK-S03（幂等 checkpoint）、EK-S05（双持久化+locality）、EK-S15（microVM 快照）、EK-S16（gVisor 快照）共享"快照-恢复"机制边，跨 2+ 独立子系统（ateworker-gvisor / ateworker-microvm / atelet 上传路径）出现
- 簇成立五条件：内聚（同一恢复管线）✓ 跨实例（两种沙箱运行时+两种 durability 模式）✓ 解释范围扩大（从"gVisor 怎么快照"到"如何设计可迁移有状态运行时"）✓ 可命名（快照-恢复三件套）✓ 可回溯（每条 EK 带证据锚）✓
- 解释范围扩大论证：三件套相互支撑——分级保真控制成本、幂等吸收分布式重试、双持久化平衡延迟与安全
- epistemic: **Pattern (L3)**，单项目验证 → Cross-project validation pending（dapr-agents 的 durable execution 为对照候选）
- links: EK-S02, EK-S03, EK-S05, EK-S15, EK-S16

## KO-S02 零信任四支柱（必沙箱 × 零凭据 × 默认拒绝网络 × 组件 mTLS）
- aggregation_rule: **R3 不变量簇** — EK-S09（egress fail-closed）、EK-S11（身份双轨）、EK-S12（零环境凭据）、EK-S13（组件 mTLS）、EK-S20（安全状态体系）经 constraint 边汇聚到同一安全不变量："actor 默认无能力，能力只在水合与放行瞬间出现"
- 簇成立五条件：内聚 ✓ 跨实例（存储/网络/身份三面独立实现同一不变量）✓ 范围扩大（从单机制到系统安全哲学）✓ 可命名（零信任四支柱）✓ 可回溯 ✓
- epistemic: **Cognitive Model (L4) pending**——"运行时代理平台的信任边界 = 默认能力为零 + 一切能力显式授予"，跨项目验证待 dapr-agents/MXC/Strands Box 对照
- links: EK-S09, EK-S11, EK-S12, EK-S13, EK-S20

## KO-S03 身份代行可区分（actor 自身 vs 组件代行分轨）
- aggregation_rule: **R4 主题簇** — EK-S11（actoridjwt 双 Kind）、EK-S12（credbundle 水合）、EK-S13（SPIFFE/组件 mTLS）覆盖"身份"主题互补维度（签发主体/载体/证书基础设施）
- epistemic: **Pattern (L3)**——"当系统需要组件代执行 actor 动作时，代行主体必须与 actor 自身主体可区分（否则无法审计/授权分离）"，Cross-project validation pending
- links: EK-S11, EK-S12, EK-S13

## KO-S04 请求停车链路（入口拦截 → 可重试 resume → 有界持有）
- aggregation_rule: **R2 因果链簇** — EK-S01（多路复用饱和场景）→ EK-S08（parking 重试）→ EK-S07（assignment 反规范化路由）形成完整链（饱和检测→停车决策→恢复路由）
- epistemic: **Pattern (L3)**——"资源瞬时不足不应立即失败给客户端：有界内部重试 + 显式背压（lot 容量）+ 超时回退"；对照：反例=立即 503（parking 文档明示其代价）
- links: EK-S01, EK-S07, EK-S08

## KO-S05 凭据外注模式（注入后丢弃 + PEP 强制）
- aggregation_rule: **R4 主题簇** — EK-S09（PEP 编译/评估）、EK-S10（注入后丢弃）覆盖"外注凭据"同一主题的强制与净化两维
- epistemic: **Pattern (L3)** → 跨项目假说 C-D05（与 dapr-agents/MXC/Strands Box 收敛）：**"凭据必须外注：沙箱内永存凭据、网关注入后即弃"**——Cross-project validation pending（MXC/Strands Box 为次级证据，非本项目内验证）
- links: EK-S09, EK-S10

## KO-S06 恢复一致性（幂等 checkpoint × torn-file 防护 × revert 保 durable）
- aggregation_rule: **R3 不变量簇** — EK-S03（幂等）、EK-S04（torn-file 防护）、EK-S06（revert 保留最后 durable）汇聚到同一不变量："**恢复动作必须可重试且状态单调安全**"（重启/超时/回滚都不能破坏可恢复性）
- epistemic: **Cognitive Model (L4) pending**——"分布式恢复的正确性 = 幂等完成标记 + 崩溃边界文件防护 + 版本单调不回退"，跨项目验证 pending
- links: EK-S03, EK-S04, EK-S06

## KO-S07 低意见运行时（lifecycle 与物理资源解耦）
- aggregation_rule: **R2 因果链簇** — EK-S18（atelet/worker 互不透明）→ EK-S01（actor→worker 多路复用）→ EK-S19（任意节点恢复）——解耦因果链完整
- epistemic: **Pattern (L3)**——"运行时若想让工作负载可迁移，必须把逻辑生命周期与物理执行单元解耦（镜像无关、Pod 可回收、恢复可跨节点）"；对照：普通 Pod=生命周期绑物理 Pod（反例）
- links: EK-S01, EK-S18, EK-S19

## KO 聚合规则矩阵（v3.1 门）
| KO | 规则 | 簇内 EK | 平均簇规模 |
|---|---|---|---|
| KO-S01 | R1 机制 | 5 | 5 |
| KO-S02 | R3 不变量 | 5 | 5 |
| KO-S03 | R4 主题 | 3 | 3 |
| KO-S04 | R2 因果链 | 3 | 3 |
| KO-S05 | R4 主题 | 2 | 2 |
| KO-S06 | R3 不变量 | 3 | 3 |
| KO-S07 | R2 因果链 | 3 | 3 |
- 聚合覆盖率：20/20 EK 被至少一个 KO 覆盖（100%）
- 无"同子系统=聚合理由"假聚合（每条均声明边类型）
