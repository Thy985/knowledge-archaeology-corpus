# 00 · Overview — PyRIT（ARCH-2026-10-03-001）

## 一句话定位
**PyRIT（Python Risk Identification Tool for generative AI）是微软 AI Red Team 官方开源框架**——把"AI 红队攻击"工程化为可组合、可编排、可评分、可持久化的攻击管线。核心命题不是"怎么攻"，而是"**攻击如何被工程化为可审计、可判定、可复现的组件流水线**"。

## 考古范围
- repository: `https://github.com/microsoft/PyRIT`
- commit: `38d7e1121bacca15299bcc22992066a6f5d73e8d`（main，2026-10-02 11:50:33 UTC）
- version: `1.2.0.dev0`（pyproject.toml）
- 规模：pyrit/ 包内 Python 190,466 行；tests/ 775 测试文件
- 本地生产版 skill：knowledge-archaeology v3.2

## 三层速览

### Project Layer（是什么）
微软官方开源 AI 红队框架：攻击者用 **Scenario（场景）→ Attack（策略）→ 组件（ConversationManager/Converter）→ PromptTarget（目标）→ Scorer（评分）→ Memory（持久化）** 的管线对生成式 AI 系统做主动风险识别。核心基础设施：通用 **Registry**（discover→introspect→construct）、强制生命周期 **Strategy**、并发 **AttackExecutor**、SQLAlchemy **CentralMemory**。

### Engineering Knowledge（怎么解决工程问题）
45 条 EK（EK Graph，六类边），覆盖：组件化攻击架构、注册表单一入口、Strategy 强制生命周期、评分唯一权威（undetermined 不弱化）、fallback 评分器 comparable 校验、attribution 可追溯链、并发执行与失败策略、动态重试、memory 标识层级表、Scenario 运行服务并发闸门等。

### Generalized Knowledge（可迁移认知）
8 个 KO（R1-R4 聚合规则），核心认知：**"攻击框架的可靠性来自可判定的判定链 + 可追溯的审计链 + 可组合的组件边界"**（攻击工具也需要自己的证据纪律）。

## 关键发现（Top 5）
1. **Score 是攻击判定的唯一权威，且 undetermined 是显式一等状态**——`attack_outcome_from_score()`（attack_strategy.py）声明式映射：undetermined score 既不读作成功也不读作失败，攻击以 undetermined 结束就是 undetermined。这是"证据纪律"在攻击侧的工程化。
2. **Registry 是架构枢纽且只有一条 add 路径**——`_discover() → register_class()` 在注册时校验 build contract（reference 参数必须映射到 wired registry），无事后清扫；`resolve_constructor_args` 统一处理简单值强制类型 + registry-reference 按名解析。
3. **Strategy 强制生命周期（setup → perform → teardown）+ 事件化**——所有攻击策略共享 `execute_with_context_async` 编排（strategy.py:310），事件 handler 可插拔，执行失败的 RuntimeError 有专用类型 `_StrategyRuntimeError`。
4. **并发执行把"参数构建失败"与"执行失败"分离治理**——AttackExecutor 用 asyncio.Semaphore 限流；`return_partial_on_failure` 决定参数构建失败是压制执行还是返回部分结果；`_raise_first_fatal_exception` 按输入顺序取首个致命异常。
5. **attribution 机制让每个 AttackResult 可反向追溯**——attribution_parent_id + objective_sha256（attack_executor.py:198-214）使聚合结果可分解回单个任务；这是"攻击审计链"的实现。

## 质量指标
- Facts/Evidence：108+
- Engineering Knowledge：45（平均出边 ≥1，游离 EK 2 条标 D）
- Flow Atlas：7 类流（Control/State/Data/Evidence/Authority/Memory/Policy）
- Generalized KO：8（aggregation_rule 100% 覆盖，R1×2 / R2×2 / R3×2 / R4×2）
- Candidates：14（含 5 跨项目假说）
- Validation：6 类审计（Truth/Coverage/Flow/Abstraction/Counterexample/Epistemic）+ Blind Reconstruction + 反例

## 与已有 Corpus 的连接
- **补攻击侧空白**：corpus 26 项目防御侧 8+（rampart/aigis/guardian/ai-protector/skillfortify/agent-governance-toolkit/owasp-agentic-skills/aprover），PyRIT 是首个攻击编排侧深度考古。
- **同厂对照**：Microsoft RAMPART（rampart 09-04 已考古）同为微软安全产品，攻防两侧同源。
- **dsh-pentest 主线**：候选卡 ai-offensive-toolchain 明确说"Garak 基线扫描 + PyRIT 深度攻击"承接 dsh-pentest 空壳——本考古提供 PyRIT 侧工程底座。
