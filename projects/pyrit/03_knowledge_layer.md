# 03 · Knowledge Layer（Generalized KO）— PyRIT（ARCH-2026-10-03-001）

> 窄尖顶：8 个 KO，每个声明 aggregation_rule（R1-R4）+ 簇内 EK + 边类型 + 解释范围扩大论证。
> 认知状态纪律：Hypothesis 进 05 Candidates；跨项目假说不写成已验证 Principle。

---

## KO-01 可判定判定链：攻击工具用"显式判定契约"保证结果可信
- **层**：L4 Cognitive Model
- **aggregation_rule**：R2 因果链簇
- **簇内 EK**：EK-14（score 唯一权威）→ EK-16（objective+auxiliary 双轨）→ EK-15（undetermined 一等状态）→ EK-17（fallback comparable 校验）；边类型 causal/contrast
- **解释范围扩大**：不止 PyRIT——任何"自动化评估/红队/基准"系统，结果可信度取决于"判定是否显式、undetermined 是否被诚实保留、回退是否校验可比"，而非"判定的模型有多强"。
- **L1-L3 底座（可回溯）**：attack_outcome_from_score（attack_strategy.py）→ score_response_async（attack_scoring.py:74）→ UndeterminedScoreError（score.py:53）→ _validate_comparable_results（fallback_scorer.py:106）。
- **反例**：无未判定路径（所有攻击都以 score→outcome 结束）；undetermined 反例是"不弱化"的正面约束证据。

## KO-02 注册即校验：扩展机制的可靠性来自"入口唯一 + 注册时验证"
- **层**：L4 Cognitive Model
- **aggregation_rule**：R1 机制簇
- **簇内 EK**：EK-08（单一路径注册）+ EK-09（统一构造）+ EK-10（注册校验两条件）+ EK-11（class/instance 分层）；边类型 mechanism
- **解释范围扩大**：插件化系统的两类退化——(a) 多入口导致校验漂移，(b) 构造时才发现 reference 解析失败。PyRIT 用"注册时一次性校验 + 统一 resolve_constructor_args"同时消灭两者。同类：mypy plugin、pytest hook 注册、Spring bean 装配。
- **L1-L3 底座**：registry.py docstring + register_class + resolution.py derive_parameters。
- **反例**：无（"no separate post-hoc sweep" 是显式设计意图，非实现缺陷）。

## KO-03 生命周期强制化：执行框架用"不可跳过的三段式"保证资源与状态正确
- **层**：L3 Pattern
- **aggregation_rule**：R1 机制簇
- **簇内 EK**：EK-02（Strategy 生命周期）+ EK-03（事件化）+ EK-42（SendContext 状态机）+ EK-37（RunService resume）；边类型 mechanism
- **解释范围扩大**：凡是"每次执行都要 setup/cleanup + 可观测 + 可恢复"的框架（测试框架、任务执行器、爬虫管线），强制生命周期 + 事件 + 恢复点三件套可复用。
- **L1-L3 底座**：strategy.py execute_with_context_async(310) + PrependedHistorySendContext 状态机 + scenario_run_service resume。
- **反例**：见 06 反例 R-01（顺序攻击 early-exit 与"必须 teardown"的张力——teardown 仍执行，符合生命周期，非反例）。

## KO-04 组合式攻击：复杂攻击 = 原子策略 × 组件 × 序列策略
- **层**：L3 Pattern
- **aggregation_rule**：R4 主题簇
- **簇内 EK**：EK-01（组件组合）+ EK-04（三类型）+ EK-05（PromptGen 分离）+ EK-06（SequentialAttack 组合）+ EK-43（对话组件）；边类型 subsystem
- **解释范围扩大**：攻击/测试/工作流系统把"执行单元"与"组合策略"解耦（组合器只做编排，单元自持判定），可无限扩展攻击形态而核心管线不变。
- **L1-L3 底座**：attack/component/ + compound/sequential_attack.py + promptgen/。
- **反例**：C-01（SequentialAttack 结果 envelope 与子结果的一等行并存——组合层与执行层共享结果语义，非矛盾）。

## KO-05 失败显式化：并发系统的失败必须"可见、可复现、可策略化"
- **层**：L4 Cognitive Model
- **aggregation_rule**：R3 不变量簇
- **簇内 EK**：EK-21（semaphore）+ EK-22（参数构建失败分离）+ EK-23（AttackExecutorResult 部分完成）+ EK-24（首异常策略）+ EK-25（动态重试）+ EK-26（RetryCollector）；边类型 constraint/causal
- **解释范围扩大**：并发批处理（评测、压测、数据管道）的"静默部分失败"是最大隐患；PyRIT 的答案——部分完成是显式状态、失败按确定性顺序抛、重试预算可运行时调、重试事件集中收集。
- **L1-L3 底座**：attack_executor.py:37-474 + exception_classes.py:74-95 + retry_collector。
- **反例**：R-02（_raise_first_fatal_exception 只抛首个异常，后续异常进 exceptions 列表——失败集合仍在，未丢失）。

## KO-06 审计链持久化：结果行自带"从哪来、属于谁"的可追溯身份
- **层**：L4 Cognitive Model
- **aggregation_rule**：R2 因果链簇
- **簇内 EK**：EK-07（attribution）→ EK-28（keyset 分页）→ EK-29（标识表镜像）→ EK-30（哈希主键）；边类型 causal
- **解释范围扩大**：审计性系统（红队报告、合规记录、实验复现）需要"结果可反向分解回产生它的任务与配置"。PyRIT 用 attribution_parent_id + objective_sha256 + 哈希标识表实现跨环境可追溯。
- **L1-L3 底座**：attack_executor.py:198-214 + memory_models.py 标识表。
- **反例**：无（见 06 中 3 成功案例之一）。

## KO-07 目标能力自描述：适配层用"能力发现"而非"硬编码分支"
- **层**：L3 Pattern
- **aggregation_rule**：R4 主题簇
- **簇内 EK**：EK-39（capability discovery）+ EK-40（TargetRequirements）+ EK-41（ModalityRouter）+ EK-44（pydantic 契约）；边类型 subsystem/mechanism
- **解释范围扩大**：多目标系统（LLM 供应商、设备、平台）适配层应让目标自报能力（capability）+ 前置条件契约（requirements），消费方按能力路由——新增目标零核心改动。
- **L1-L3 底座**：discover_target_capabilities.py + target_requirements.py + modality_router.py。
- **反例**：C-02（CapabilityName 枚举集中 vs 目标自描述——能力词汇表仍是中心化枚举，见 05）。

## KO-08 配置驱动初始化：系统启动 = 配置声明的组件装配
- **层**：L3 Pattern
- **aggregation_rule**：R4 主题簇
- **簇内 EK**：EK-13（Initializer）+ EK-12（注册表族）+ EK-38（Scenario 配置解析）+ EK-36（规模预估）；边类型 subsystem/causal
- **解释范围扩大**：复杂框架用"配置声明的初始化器列表"替代硬编码装配——用户按需启用组件族（target/converter/scorer），装配逻辑集中在 Initializer。
- **L1-L3 底座**：.pyrit_conf_example Initializers 段 + initializer_registry.py。
- **反例**：无（初始化顺序由配置列表决定，非隐式依赖注入）。

---

## KO 聚合规则覆盖矩阵
| KO | 规则 | 簇内 EK 数 | 可回溯 | 解释范围扩大 |
|---|---|---|---|---|
| KO-01 | R2 因果链 | 4 | ✓ | ✓（判定契约） |
| KO-02 | R1 机制 | 4 | ✓ | ✓（注册即校验） |
| KO-03 | R1 机制 | 4 | ✓ | ✓（强制生命周期） |
| KO-04 | R4 主题 | 5 | ✓ | ✓（组合式攻击） |
| KO-05 | R3 不变量 | 6 | ✓ | ✓（失败显式化） |
| KO-06 | R2 因果链 | 4 | ✓ | ✓（审计链） |
| KO-07 | R4 主题 | 4 | ✓ | ✓（能力自描述） |
| KO-08 | R4 主题 | 4 | ✓ | ✓（配置驱动装配） |

- 聚合规则覆盖率 100%；无"同子系统=聚合理由"假聚合（每条 KO 均有机制/因果/不变量/主题知识性依据）。
- 升维纪律：KO-01/02/05/06 为 L4（稳定关系），其余 L3（可复用模式）；无 L5 方法论——单项目证据不足以支撑方法论级升维（跨项目验证留给 05）。
