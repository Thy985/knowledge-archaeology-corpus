# 05 Candidates — 未确认内容（Hypothesis / 暂定模式 / 跨项目假说）

> 与 KO 严格区分：以下内容证据不足以升维，保留 Hypothesis 状态与验证路径。

## C-01 跨项目假说：记忆写路径判定门 vs 记忆系统实现
- **状态**: Hypothesis（Cross-project validation pending）
- **内容**: "主流 agent 记忆系统（mem0/letta-code/LangGraph 原生 checkpoint）的写路径缺少判定门，毒化窗口普遍存在"
- **当前证据**: 本项目 write 管线五阶段（EK-02）；corpus 中 mem0/letta-code 考古未见等价判定门
- **缺失证据**: 未对 mem0/letta-code 源码做写路径对比审计；未验证 LangGraph checkpoint 写路径
- **验证路径**: 下一轮针对记忆系统考古时补"写路径判定门存在性"清单项

## C-02 暂定模式：安全工具"边界可见性"是普遍工程文化还是本项目特色
- **状态**: Hypothesis
- **内容**: "安全工具高比例缺乏边界声明（digest 不验证/不认证/数字附语料），本项目是少数派"
- **当前证据**: 本项目三处主动边界声明（EK-01/20/23）+ compliance-mapping §0；无对照样本
- **缺失证据**: 未抽样同类工具（guardian/ai-protector/aigis）的边界声明实践
- **验证路径**: corpus 防御品类对比轮

## C-03 不确定结论：92.5% recall 对生产部署的代表性
- **状态**: Observation（数据来自仓库自报，非独立复测）
- **内容**: compliance-mapping 明示 55 自编 cases、非自适应对手度量；README 表 92.5%（0.2.2）；AMSB 21 scenarios 独立于 55-case 套件
- **不确定性**: 两套数字口径不同（55-case 检出率 vs 21-scenario 基准分）；未在本轮复跑
- **验证路径**: 复跑 `python benchmarks/security_benchmark.py` + `amg-bench` 对比（本轮未跑——时间盒限制，列入 run 缺口）

## C-04 暂定模式：verified=True 单布尔门槛的充分性
- **状态**: Hypothesis
- **内容**: "promote 到 verified_preference 仅需 verified=True 布尔，可能是可绕过的单因子门"
- **当前证据**: PromotionRules.requires_verification → guard.promote 检查 verified（guard.py:195-215）
- **缺失证据**: 无 verified_by 强度校验；无多因子/凭证绑定
- **验证路径**: 攻击视角测试（伪造 verified=True 的调用方）

## C-05 跨项目假说：AMSB 评分模型可迁移到其他安全中间件基准
- **状态**: Hypothesis（Cross-project validation pending）
- **内容**: "canary 存活判定 + FP 惩罚 + 等级封顶的组合适用于通用安全中间件基准"
- **当前证据**: EK-24/25/26 三构件在 AMSB 实现
- **缺失证据**: 未在其他项目（如 guardrails 类）实测
- **验证路径**: 用 AMSB 语料对 guardian/ai-protector 跑 adapter（bench/adapters/third_party.py 已留接口）

## C-06 观察：OWASP 工程化四件套中的定位
- **状态**: Observation
- **内容**: 10-07 雷达称 OWASP 从"Top10 列表"升级为"可操作工程件四件套"（AST10 v2 / Cheat Sheet / AIVSS / ACS）；本仓库是其中"ASI06 参考实现"承载者
- **证据**: 仓库 README/CHANGELOG 提及 ASI06/MITRE ATLAS（AML.T0080.000）；雷达增量文本
- **不确定**: 四件套与本仓库的官方隶属关系以 OWASP 官网为准（未核验官网）

## C-07 不确定：受信佐证集的默认值影响面
- **状态**: Observation
- **内容**: #124 后默认仅 SYSTEM 可衰减自增强冷却；部署若未配置 trusted_source_classes，USER_INPUT/EXTERNAL_TOOL 佐证完全不衰减——保守但可能误伤合法独立证据场景
- **证据**: self_reinforcement.py 默认 `_DEFAULT_TRUSTED_SOURCE_CLASSES={SYSTEM}`；#124 commit 描述
- **不确定**: 实际误报率未测量（bench benign 6 例含相关场景但未独立标注）

## C-08 观察（Reconciliation 补充，IF-3）：regex 检测可被词形插入规避——已知族内对抗变体实测
- **状态**: Observation（AMSB 实测证据）
- **内容**: hardened 配置下唯一 breach 为 `persist-directive-evasion`（"Directive promotion with an adjective inserted to evade fixed patterns"）——词形插入（形容词/修饰语）规避固定 regex 的已知族内变体；同理 prompt_injection/sensitive_data regex 均可能被词形改写绕过
- **证据**: AMSB 21 语料实测 hardened=1/15 breach（本 run 独立复跑）；scenarios.py persist-directive-evasion 定义
- **含义**: 与 KO-04 的"regex 快速层"互补——regex 层的对抗边界在仓库内即有实测反例；ML 层（默认关闭）是设计中的弥补手段
