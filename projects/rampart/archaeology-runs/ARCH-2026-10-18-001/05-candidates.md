# 05 · Candidates（未验证假设 / 跨项目假说）— RAMPART refresh（ARCH-2026-10-18-001）

> 纪律：不能确认的内容留在候选。Hypothesis ≠ Fact；Cross-project Candidate ≠ 已验证 Principle。

## C-R01 · trace runner 将成为唯一执行路径（Hypothesis）
- **状态**: Hypothesis（未实现验证）
- **内容**: `core/trace.py` 的共享 runner 设计为统一底座，但 XPIA/SingleTurn 尚未迁移。假设未来所有执行策略收敛到 `run_trace_async`（迁移信号：docs/results-and-reporting.md "Existing prefix-evaluated strategies leave these fields as None until their follow-up migration"）。
- **当前证据**: trace.py 存在 + result.py 字段就位 + docs 明示迁移意向
- **缺失证据**: 尚无任何策略实际调用 run_trace_async（git grep 确认 _xpia/_single_turn 未引用）
- **验证路径**: 观察 RAMPART 后续 PR：XPIA/SingleTurn 迁移后 status 计算如何消费 final_trace_evaluation

## C-R02 · 双视图隔离（raw/turns）是否有性能/语义代价（Hypothesis）
- **状态**: Hypothesis
- **内容**: turns 与 raw_turns 双列表在每 turn 复制（raw_turn → replace → turns append + raw_turns append），长 trace 下内存双倍；且 `_matches_final_trace_context` 每次终局评测逐 turn 比较。假设对长会话（max_turns 大）有可测开销。
- **当前证据**: trace.py:143-171（双 append）
- **缺失证据**: 无基准/性能测试对比
- **验证路径**: 长 trace benchmark；或在 RAMPART issues 中观察是否出现性能讨论

## C-R03 · 无 upcaster 哲学可能阻碍生态采用（Cross-project Hypothesis）
- **状态**: Cross-project Hypothesis（跨项目假说）
- **内容**: RAMPART 选择"无 v1 decoder/upcaster，迁移须显式"——这在单一主控方（Microsoft）下可行；对第三方已用 v1 的消费者，无 upcaster 意味着必须重写导入逻辑。假设生态采用后会出现迁移痛点的 issue/讨论。
- **当前证据**: schemas/trace-compatibility.json migration_note（明示无 upcaster）
- **缺失证据**: 尚无第三方消费者证据（项目 Alpha 阶段，PyPI 采用度低）
- **验证路径**: 观察 RAMPART issues 中 v1→v2 迁移相关反馈；与 langfuse/mem0 等序列化治理对照（同主题不同策略）

## C-R04 · RAMPARTner skill 的 conversation contract 是否可复制（暂定 Pattern 候选）
- **状态**: Pattern 候选（单项目证据）
- **内容**: RAMPARTner 把 onboarding 固化为：固定开场 + source choice 纪律（main vs release 显式询问）+ evaluation contract 先于实现 + teach-with-every-response + local-extension vs framework-feature 区分。这些元素与 webwright skill 的 plan/CP/self-verify 契约、与 knowledge-archaeology 自身的 contracts 体系同构——"引导性 skill 需要对话契约"可能是跨项目模式。
- **当前证据**: rampart/skills/rampartner/SKILL.md:1-60（单项目）
- **缺失证据**: 至少需第二个独立项目使用同类契约并验证效果
- **验证路径**: 对比 webwright（corpus 已考古）的 conversation/workspace contract；若出现第三个项目，升 L3

## C-R05 · eval_purpose 扩展可能（Hypothesis）
- **状态**: Hypothesis
- **内容**: `EvaluationPurpose` 目前仅 STOP_CHECK 一个成员。假设未来增加（如 FINAL_CHECK、BENCHMARK）以区分更多评估来源——字段的 Optional 设计（None=未记录）正是为此预留。
- **当前证据**: types.py:267-276（枚举仅 STOP_CHECK）+ Turn.eval_purpose Optional
- **缺失证据**: 无任何意图声明
- **验证路径**: 观察后续 PR 是否扩展枚举

## C-R06 · trace schema v2 与 xdist v2 版本号分轨（Observation）
- **状态**: Observation（不是结论）
- **内容**: trace schema 版本（rampart.trace.v2）与 xdist schema 版本（rampart.xdist.v2）是**两个独立版本空间**，各自有兼容性机制（trace 有显式决策文件；xdist 有 SCHEMA_VERSION 字符串校验）。二者是否会合并/漂移待观察。
- **当前证据**: serialization.py:52 vs _xdist.py:58
- **缺失证据**: 无合并意图证据
- **验证路径**: 观察后续 PR

## C-R07 · method marker 遮蔽（shadow）语义是否为通用 pytest 行为（Cross-project Hypothesis）
- **状态**: Cross-project Hypothesis（Auditor F6 引入）
- **内容**: `get_closest_marker` 使 method 级 `@pytest.mark.trial(n=2)` **遮蔽** class 级 `trial(n=5, threshold=0.9)`（不合并，threshold 回落到默认 1.0）——test_trial_integration.py:72-96 锁定。假设这是 pytest marker 解析通用语义（非 RAMPART 定制），其他框架（含用户 E2E-CLI）复用 marker 声明时应知晓 shadow 而非 merge。
- **当前证据**: 单项目测试锁定（test_trial_integration.py）
- **缺失证据**: pytest 官方文档/其他项目交叉验证
- **验证路径**: 查阅 pytest get_closest_marker 文档；或对比其他 pytest 插件（pytest-timeout 等）的 marker 继承语义
