# 03 · Knowledge Layer（Generalized KO）— RAMPART refresh（ARCH-2026-10-18-001）

> 窄尖顶。每个 KO 声明 aggregation_rule（R1-R4 簇）+ 簇内 EK + 边类型。本文件为 **refresh 新增/升级 KO**；09-04 基线 KO-01~07（corpus 03_knowledge-layer.md）保持有效。
> Epistemic 纪律：单项目证据 → Pattern/Model 标 `(RAMPART validated; cross-project pending)`；Hypothesis 进 05 Candidates。

---

## KO-R01 · 终局证据与执行证据分离：判定不得由过程推断
- **层**: L4 Cognitive Model ｜ **aggregation_rule**: **R2 因果链簇**（EK-R02→EK-R03→EK-R08：分层证据→版本化持久化→无推断迁移）
- **簇内 EK**: EK-R02(online≠final) →EK-R03(信封持久化) →EK-R08(无 upcaster/absence 语义)
- **一句话**: 验证系统的**过程证据**（在线 stop-check）与**终局判定证据**（final-trace evaluation）必须分离存储，且任何代码路径都不得从过程数据推断终局结论——"未记录"是合法状态，不是缺失值。
- **实例**: stop_when 的 STOP_CHECK 标注是执行期证据；`Result.final_trace_evaluation` 是终局证据；codec 永不从 status/turns 推断（docs trace-schema.md:62-65）。
- **解释范围扩大**: 任何长流程验证/审计系统——过程日志（发生了什么）与结论（判定结果）分路，结论只能由判定器产生。
- **证据强度**: S3/S4（实现 + test_trace.py/test_serialization_evaluations.py 锁定 absence 语义）
- **scope**: applies_when 判定链长且过程可观测；does_not_apply_when 单步确定性判定（推断无害时）
- **epistemic**: Cognitive Model（RAMPART validated; cross-project pending）

## KO-R02 · 版本化契约治理：兼容性必须是显式决策，不是隐式容错
- **层**: L4 Cognitive Model ｜ **aggregation_rule**: **R3 不变量簇**（EK-R07+EK-R08+EK-R10 汇聚到"契约指纹链不可断裂"不变量）
- **簇内 EK**: EK-R07(compatibility gate) ↔EK-R08(无 upcaster) ↔EK-R10(strict JSON-mode) —— constraint 边汇聚
- **一句话**: 数据契约的演化必须走"**指纹→显式决策→迁移说明**"管道；禁止静默 relabel、禁止隐式 upcast、禁止发明缺失值（provenance 需真实来源）。
- **实例**: v1→v2 因 PopulationRef.id 非空要求而 new-major；v1 schema 保留；迁移须显式转换；空 population id 须合法 provenance 或再生。
- **解释范围扩大**: 适用于所有跨进程/跨版本数据交换（序列化格式、API schema、config 契约）——"兼容"是决策产物，fingerprint 使其可审计。
- **证据强度**: S4（test_trace_compatibility.py 锁定 INITIAL/COMPATIBLE/NEW_MAJOR 决策约束 + 指纹链校验）
- **scope**: applies_when 契约有跨版本消费者；does_not_apply_when 一次性内部数据（无历史契约负担）
- **epistemic**: Cognitive Model（RAMPART validated; cross-project pending）

## KO-R03 · 判定者与被判定历史隔离（raw view）：评估输入不得携带框架标注
- **层**: L4 Cognitive Model ｜ **aggregation_rule**: **R1 机制簇**（EK-R01+EK-R04 共享"视图分离"机制，跨 trace runner + result 两子系统；同源 09-04 EK-23 judge 降级）
- **簇内 EK**: EK-R01(turns/raw_turns 双视图) ↔EK-R04(复用基于 raw_turns 快照) ↔EK-23(judge malformed→UNDETERMINED, 09-04)
- **一句话**: 让判定器消费**从未被框架标注过**的原始轨迹——否则评估结果会自我污染（看到"上一轮评估"而调整本轮判定）。
- **实例**: `raw_turns` 永不带 eval_result/eval_purpose；evaluator context 由 raw_turns 构建；driver/report 用 turns。
- **解释范围扩大**: 所有"用 AI 评估 AI/系统"的回路——训练数据、评测输入、审计日志都应隔离注入的元数据，避免评估者读到自己的标注。
- **证据强度**: S3/S4（实现 + test_trace.py）
- **scope**: applies_when 评估器可观察到历史记录且框架会写标注；does_not_apply_when 评估器无状态或历史不可见
- **epistemic**: Cognitive Model（RAMPART validated; cross-project pending）

## KO-R04 · schema 策略与 live 类型分离：序列化约束不该污染运行时
- **层**: L3 Pattern ｜ **aggregation_rule**: **R1 机制簇**（EK-R09+EK-R06 共享"adapter-local schema 副本"机制，跨 serialization + result）
- **簇内 EK**: EK-R09(adapter-local schema) ↔EK-R06(population bounds) —— 同一"副本施加策略"机制
- **一句话**: 当传输格式需要比运行时更严的约束时，把策略施加在**序列化 schema 副本**上，而不是改 live dataclass——运行时保持宽松，线上保持严格。
- **实例**: _schema.py 对 adapter 的 schema 副本加 strict/allow_inf_nan=False/PopulationRef bounds；live 构造不受影响。
- **证据强度**: S4（test_serialization_properties.py 验证 schema 策略）
- **scope**: applies_when 同一类型有运行时与传输两种约束面；does_not_apply_when 运行时即传输（无分离必要）
- **epistemic**: Validated Pattern（RAMPART validated; cross-project pending）

## KO-R05 · 参数类型纪律：显式拒绝 bool/非有限值（类型陷阱防御）
- **层**: L3 Pattern ｜ **aggregation_rule**: **R1 机制簇**（EK-R06+EK-R14+EK-R10 共享"精确类型判断"机制，跨 population/trial/schema 三子系统）
- **簇内 EK**: EK-R06(validate_population_size 用 `type(value) is not int`) ↔EK-R14(trial n 拒绝 bool) ↔EK-R10(strict no coercion)
- **一句话**: 校验数值参数时，**显式拒绝 bool**（`isinstance(True, int)` 为真）与**非有限值**（inf/nan）——错误输入在边界被拦截，而不是在深处产生难查的语义错误。
- **证据强度**: S4（test_serialization_properties / _trial resolve 测试）
- **scope**: applies_when 参数可被外部/序列化输入驱动；does_not_apply_when 内部常量参数
- **epistemic**: Validated Pattern（跨项目常见——同类：任何 CLI 参数解析）

---

## KO 聚合规则矩阵（refresh 子集）
| KO | 规则 | 簇内 EK | 边 | 解释范围 |
|---|---|---|---|---|
| KO-R01 | R2 因果链 | EK-R02→R03→R08 | causal | 过程证据≠终局结论 |
| KO-R02 | R3 不变量 | EK-R07+R08+R10 | constraint 汇聚 | 契约指纹链 |
| KO-R03 | R1 机制 | EK-R01+R04+EK-23 | mechanism | 评估输入隔离 |
| KO-R04 | R1 机制 | EK-R09+R06 | mechanism | schema/live 分离 |
| KO-R05 | R1 机制 | EK-R06+R14+R10 | mechanism | 类型纪律 |

> 说明：refresh KO 均锚定在 EK-R 增量子图上，且与 09-04 KO 底座（KO-01 诚实不确定、KO-03 observability 上界）构成认知连续体——RAMPART 6 周演进的方向是"把不确定与证据纪律系统化"。
