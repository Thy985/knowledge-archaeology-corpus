# Knowledge Layer — MXC Generalized KO（L3/L4，6 条）

> 每个 KO 声明 aggregation_rule（R1-R4 簇）+ 解释范围扩大论证；L3 Pattern / L4 Cognitive Model；无 L5 硬凑。
> 未跨项目验证的原则一律标 `Cross-project validation pending`。

## KO-M01 · fail-closed 是链条不是口号（L3 Pattern · R2 因果链簇）
- **aggregation_rule**: R2 因果链簇——EK-M02(closed JSON)→EK-M18(后端拒绝未支持策略)→EK-M03(网络默认 deny)→EK-M04(FS fail-closed)→EK-M13(telemetry fail-closed)；沿 causal/mechanism 边成链
- **簇内 EK**: EK-M02/M03/M04/M13/M18
- **解释范围扩大**: 单项目机制集合 → "每个可失败点显式闭合"的模式；对照 strands-box（DenyReason::PolicyPending/InternalFault + 引擎 pin）跨项目证据
- **反例（预算内）**: permissiveLearningMode 显式弱化 deny-by-default（EK-M08，audit 模式即例外）；Seatbelt profileOverride 绕过生成策略（EK-M12）——**反例说明 fail-closed 是默认态而非不变量**
- **epistemic**: Pattern（跨项目对照 strands-box 成立；MXC 内部 S4 测试验证）

## KO-M02 · 契约/运行时/授权三轴分离（L4 Cognitive Model · R3 不变量簇）
- **aggregation_rule**: R3 不变量簇——EK-M14(三轴权威表)+EK-M05(注册表)+EK-M06(experimental 门)+EK-M07(引擎路由) 汇聚到同一不变量："哪个契约被接受"≠"哪个后端运行"≠"主机能强制什么"
- **簇内 EK**: EK-M05/M06/M07/M14
- **一句话稳定关系**: **契约合法性、运行时授权、主机强制是三个独立决策面——任何一面都不能授权另一面**
- **解释范围扩大**: 从 MXC 版本化治理 → 一般系统（schema 版本 vs 实验特性 vs 平台能力）
- **反例**: dev 契约可变（1.1.0-alpha 无 notice 变化）vs published 不可变——可变/不可变是同一三轴内的状态位而非反例
- **epistemic**: Cognitive Model（`Cross-project validation pending`——需第二项目验证）

## KO-M03 · 能力边界诚实（L3 Pattern · R4 主题簇）
- **aggregation_rule**: R4 主题簇——EK-M10(proxy 限制)+EK-M11(IsolationSession 强制全 allow)+EK-M12(Seatbelt 逃生门)+EK-M16(WSLc 双态)：同一主题"后端能力差异"，覆盖互补维度
- **簇内 EK**: EK-M10/M11/M12/M16
- **模式**: **不能强制就拒绝，不静默假装**——IsolationSession 显式要求三全 allow（空 network 拒绝）、WSLc 拒绝混合姿态、Seatbelt 显式逃生门、proxy 明示"不能保证任意直连被拦"
- **解释范围扩大**: 隔离/沙箱选型的通用纪律——能力不足的后端必须显式声明而非默认假装安全
- **epistemic**: Pattern（MXC 内多后端一致验证）

## KO-M04 · Learning→Enforcement 渐进路径（L3 Pattern · R1 机制簇）
- **aggregation_rule**: R1 机制簇——EK-M08(双 capability)+EK-M09(DenialAnalyzer)+EK-M19(策略发现) 共享"先观察后强制"机制且跨 ≥2 子系统（policy 输入 / capture / 分析）
- **簇内 EK**: EK-M08/M09/M19
- **模式**: deny-and-record（containment 不变，仅记录）→ 分析有界去重 → 生成最小权限候选（plm denials.json + compatibility generator）——渐进放权的安全路径
- **解释范围扩大**: 最小权限落地的通用流程（对照 Evaluation 周"执行观测"方法）
- **epistemic**: Pattern

## KO-M05 · 单一引擎路由 + 显式扩展成本（L4 Cognitive Model · R2 因果链簇）
- **aggregation_rule**: R2 因果链簇——EK-M07(resolve_runner 收敛)→EK-M05(ProbeCoverage 强制声明)→EK-M06(experimental 门)：扩展→声明→门控的因果链
- **簇内 EK**: EK-M05/M06/M07
- **一句话稳定关系**: **让"加一个后端"的决策成本显式化（必须声明探测覆盖、必须过实验门），比事后审计更省力**
- **解释范围扩大**: 插件/驱动/后端的注册模式一般化
- **epistemic**: Cognitive Model（`Cross-project validation pending`）

## KO-M06 · 治理代码化（L4 Cognitive Model · R4 主题簇）
- **aggregation_rule**: R4 主题簇——EK-M17(测试文化)+EK-M18(架构不变量)+EK-M20(单一契约) 覆盖"开发者协作如何被文件化约束"
- **簇内 EK**: EK-M17/M18/M20
- **一句话稳定关系**: **把评审标准（行为覆盖、不发明证据、复用正常路径、拒绝未支持策略）写进仓库文件，让 agent/协作者被同一组不变量约束**
- **解释范围扩大**: 对照 agent-aegis（AGEF/AGP 自述标准）与 strands-box（AGENTS.md 权限矩阵）——治理文件是跨项目可比较的"自我约束证据面"
- **反例**: 文档约束无强制（AGENTS.md 是 guideline 非 lint）——"代码化"是文化非机制
- **epistemic**: Cognitive Model（跨项目证据：strands-box/agent-aegis 对照，`Cross-project validation pending`）

## KO 聚合规则矩阵
| KO | 规则 | 簇内 EK | 边类型 | 可命名 | 可回溯 |
|----|------|---------|--------|--------|--------|
| KO-M01 | R2 | M02/M03/M04/M13/M18 | causal/mechanism | ✅ | ✅ |
| KO-M02 | R3 | M05/M06/M07/M14 | constraint | ✅ | ✅ |
| KO-M03 | R4 | M10/M11/M12/M16 | subsystem/contrast | ✅ | ✅ |
| KO-M04 | R1 | M08/M09/M19 | mechanism | ✅ | ✅ |
| KO-M05 | R2 | M05/M06/M07 | causal | ✅ | ✅ |
| KO-M06 | R4 | M17/M18/M20 | subsystem/mechanism | ✅ | ✅ |

## 层级分布
- L3: 3（KO-M01/M03/M04）；L4: 3（KO-M02/M05/M06）；L5: 0（不硬凑方法论）
- 无"同子系统=聚合理由"的假聚合；聚合规则覆盖率 100%
