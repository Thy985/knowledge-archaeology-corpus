# 03 Knowledge Layer — Generalized KO（窄尖顶，L3/L4/L5）

> 每个 KO 声明 aggregation_rule（R1-R4）+ 簇内 EK + 反例预算 + Epistemic 状态。升维论证：解释范围必须大于单项目。

---

## KO-01 决策管线模式：provenance→detection→decision→commit 四段式记忆写入
- **aggregation_rule**: **R2 因果链簇**（EK-02 → EK-05 → EK-07 → EK-03，沿 causal 边成链）
- **簇内 EK**: EK-02（write 五阶段）/ EK-03（read 双保险）/ EK-05（异常 fail-open）/ EK-07（不可检查=finding）
- **模式（L3）**: 记忆安全中间件的写入路径是"先证明来源、再检测、再决策、最后提交"的受控管线，提交是管线的终点而非起点；读取路径反向先验证完整性再交付内容
- **认知模型（L4）**: **对记忆的写入是状态变更，状态变更必须先过判定门**——任何"先写后查"或"写时无判定"的记忆层都存在毒化窗口
- **反例预算**: ①permissive 默认下 BLOCK 分支不触发（管线仍在，判定降级为记录）——管线存在≠拦截生效；②REDACT 分支改写值后提交（判定后提交依然成立）；③QUARANTINE 不入 store（管线终点变"隔离区"）
- **Epistemic**: Pattern（本项目强证据；跨项目验证 pending——对比 mem0/letta 写路径无判定门可做）

## KO-02 不可检查内容 = 发现而非静默（深度边界不变量）
- **aggregation_rule**: **R3 不变量簇**（EK-06/EK-07/EK-34 经 constraint 边汇聚到同一不变量）
- **不变量**: 检测器读不到的内容必须成为可见 finding；安全边界（截断/异常/超限）不能变成静默盲区
- **认知模型（L4）**: **安全工具的"检查范围"本身就是安全属性——截断、降级、跳过的每一条路径都必须可见**；一次"修崩溃"的修复（#135）若无二次修复（#151）会制造比原缺陷更隐蔽的绕过
- **反例预算**: ①detector 异常路径（#138）是同一不变量的另一实例（fail-open 但发事件）；②Snapshot.digest 不验证（EK-20）是"边界不可见"的反例——快照层主动声明不检查
- **Epistemic**: Principle（本项目两独立修复史 + 事件路径共同支撑，标 `Cross-project validation pending`）

## KO-03 fail-open 必须带可见性（检测降级不变量）
- **aggregation_rule**: **R3 不变量簇**（EK-05/EK-22/EK-23 汇聚）
- **不变量**: 检测器故障、来源不明、匹配降级——任何"没拦住"都必须在事件流里留痕；guard.events 干净 ≠ 检测器全跑了
- **认知模型（L4）**: **fail-open 是安全工具的合法状态，但静默 fail-open 是缺陷**——错误与降级要进入同一观测通道
- **反例预算**: ①事件 handler 抛异常被吞（_emit 中 try/except）——观测通道自身可断，属已知边界；②source_class 可伪造——"留痕"不等于"可信"
- **Epistemic**: Principle（本项目 + 通用安全工程共识，标跨项目 pending）

## KO-04 分层防线族：regex 快速层 → 语义分类层 → ML 层 → 完整性层
- **aggregation_rule**: **R1 机制簇**（EK-12/EK-13/EK-14/EK-18/EK-19 共享检测机制、跨 ≥2 独立子系统）
- **模式（L3）**: 内容检测按"成本-准确率"分层：regex 先拦（µs 级），语义分类管信任（provenance/晋升），ML 管混淆（可选 torch），完整性管篡改（SHA-256）——各层失败由下一层兜底
- **认知模型（L4）**: **对同一威胁（记忆毒化）的防御必然是多层异构的——单层检测是绕过面，不是防线**
- **反例预算**: ①ML 层默认关闭（需 [ml] extra）——默认部署实际是 regex+语义+完整性三层；②完整性层只覆盖 immutable 键，普通键无基线；③**默认配置边界（Reconciliation 收紧，IF-1）**：默认 7 检测器套件**不含 memory_persistence_injection**——ASI06 核心威胁（延迟生效持久化指令）在官方推荐 enforcing 配置（strict）下不设防；AMSB 实测 strict=4/4 memory_persistence breach（71.4 分 D），hardened（+persistence 检测器）才收敛到 1/15（93.9 分 A）。"分层防线"在默认配置下覆盖 7 个威胁族，persistence 注入需显式 opt-in
- **Epistemic**: Pattern（分层检测在 IDS/防毒软件中普遍，本项目是 agent 记忆域实例）

## KO-05 信任梯度 = 晋升图（untrusted 不可静默变 trusted）
- **aggregation_rule**: **R1 机制簇**（EK-16/EK-17/EK-15 共享"信任提升受门控"机制，跨 classification/detectors 两子系统）
- **认知模型（L4）**: **记忆条目的可信度必须随来源显式标注，且信任提升只能经显式验证边**——检索文本/工具输出（untrusted）永远不能静默变成策略/已验证偏好；自增强衰减只认受信佐证（默认仅 system）
- **反例预算**: ①source_class 由调用方自报（EK-23）——门控在诚实 provenance 假设下成立，恶意调用方可谎报；②verified=True 是单一布尔，无多因子
- **Epistemic**: Cognitive Model（本项目强证据；与 agent 记忆域普遍实践一致，标跨项目 pending）

## KO-06 诚实的安全边界声明（permissive 默认 / 不认证 / 不验证 / 数字附语料）
- **aggregation_rule**: **R4 主题簇**（EK-01/EK-20/EK-23/EK-31 覆盖"边界声明"互补维度）
- **模式（L3）**: 安全工具主动声明自己不做什么：默认只检测不拦截、provenance 不认证、快照 digest 不验证、基准数字附语料且声明不测自适应对手
- **认知模型（L4）**: **安全保证的效力与其边界声明的诚实度成正比——不声明边界的"拦截/验证/准确率"承诺是可被攻破的承诺**
- **反例预算**: ①README benchmark 表（92.5% recall）无边界声明，compliance-mapping 有——对外叙事不统一；②CHANGELOG 可能存在未声明行为
- **Epistemic**: Principle（本项目独特的高诚实度工程文化；跨项目对比：多数安全工具不声明 digest 不验证）

## KO-07 事件驱动观测闭环：SecurityEvent → SIEM/OTel → 合规映射
- **aggregation_rule**: **R2 因果链簇**（EK-22 → EK-30 → EK-31）
- **模式（L3）**: 每个 guard 决策产出一结构事件（含外部收据 URI 指针），经 handler/OTel span 进 SIEM，再映射 NIST AI RMF / EU AI Act 子类目——"检测→观测→合规"一条链
- **认知模型（L4）**: **安全决策的可审计性是合规能力的原材料——没有事件流，任何安全控制都无法在监管框架中主张有效性**
- **反例预算**: ①receipt_uri 仅指针，仓库内无签名验证逻辑；②compliance 映射是文档声明非自动化验证
- **Epistemic**: Pattern（事件驱动观测为通用安全工程模式，本项目给 agent 记忆域实例）

## KO-08 可判定安全基准：canary 存活判定 + FP 惩罚 + 等级封顶
- **aggregation_rule**: **R4 主题簇**（EK-24/EK-25/EK-26 覆盖 AMSB 评分体系互补维度）
- **模式（L3）**: 安全基准要"可判定、不可游戏"：单一确定性规则（canary 是否存活进 recalled value）、benign 反过拦截惩罚（全拒不是防御）、单 critical breach 封顶等级（防高均值掩盖灾难）、自提交行显式标记
- **认知模型（L4）**: **安全系统的基准分必须同时惩罚"漏防"与"过度防御"——只优化防御率会诱导不可用的全拒系统**
- **反例预算**: ①21 个 scenario 是自编语料（EP：对未知攻击族无保障）；②canary 规则只测"回读毒化"，不测"写路径已拦截但回读仍毒"的中间态（harness 注释承认 write 拦截即算 defended，read 出毒才算 breach）
- **Epistemic**: Pattern（模型无关基准设计可迁移；对标 OWASP/SSL Labs 评分哲学）

---

## 聚合矩阵
| KO | rule | 簇规模 | 解释范围扩大论证 | 反例数 |
|---|---|---|---|---|
| KO-01 | R2 | 4 EK | 记忆写入管线的通用骨架 | 3 |
| KO-02 | R3 | 3 EK | 检查范围=安全属性的通用原则 | 2 |
| KO-03 | R3 | 3 EK | fail-open 可见性通用不变量 | 2 |
| KO-04 | R1 | 5 EK | 分层检测通用模式（agent 记忆域实例） | 2 |
| KO-05 | R1 | 3 EK | 信任梯度晋升门通用模型 | 2 |
| KO-06 | R4 | 4 EK | 边界声明的工程文化模式 | 2 |
| KO-07 | R2 | 3 EK | 检测→观测→合规链条 | 2 |
| KO-08 | R4 | 3 EK | 可判定不可游戏基准设计 | 2 |

## L1→L5 纵向链示例（每一条成立）
```
EK-19（canonical_serialize→SHA-256 基线，L1）
  → EK-03（read 先 verify，L2）
  → KO-02（不可检查内容=发现的深度边界，L3）
  → L4 认知模型（检查范围本身是安全属性）
  → L5 方法论：安全工具的截断/降级/跳过路径必须显式可见（可操作准则）
```
