# 03 · Knowledge Layer（Generalized KO）

> 窄尖顶。每个 KO 必须声明 `aggregation_rule`（R1-R4 簇）+ 簇内 EK + 边类型，并满足簇成立五条件（内聚性/跨实例性/解释范围扩大/可命名/可回溯）。
> 反退化铁律：聚合理由不是"都属于某子系统"。每条 KO 声明 `scope.applies_when` / `scope.does_not_apply_when`（deepseek-harness KO-03 教训）。
> Epistemic：单项目证据 → Pattern/Model 标 `(RAMPART validated; cross-project pending)`；不冒充 Principle。

---

## KO-01 · 诚实的不确定：三值判定是可验证 agent 系统的可信度基础
- **层**：L4 Cognitive Model ｜ **aggregation_rule**: **R1 机制簇**（EK-01/EK-02/EK-06/EK-23，共享"三态不确定"机制，跨 core/types + evaluator + result + llm_judge 4 子系统）
- **簇内 EK**：EK-01(observability 声明) ↔EK-02(三态) ↔EK-06(undetermined_operands) ↔EK-23(judge 降级)
- **一句话**：在判定 agent 行为时，**"无证据"与"证据为无"必须结构性区分**；不确定不是失败态，而是值得显式报告的判定结果。
- **实例**：RESPONSE_ONLY+零工具→UNDETERMINED（EK-05）；judge malformed→UNDETERMINED（EK-23）；复合评估中未确定操作数如实上浮（EK-06）。
- **解释范围扩大**：从"RAMPART 怎么处理看不到的工具调用"扩大到"任何依赖黑盒观测判定 agent 行为的验证系统"。
- **证据强度**：S4（单测锁定 test_xpia observability 降级 + test_payload_store_security + test_llm_judge 降级）
- **scope**：
  - applies_when：判定依据来自外部黑盒观测（LLM 输出、工具调用、side effect）且观测通道可能不完整
  - does_not_apply_when：判定依据来自确定性白盒（单测断言、静态代码）；此时三态判定会掩盖真实失败
- **epistemic**：Cognitive Model（RAMPART 内验证充分；cross-project pending）

## KO-02 · 评估器极性分离：条件检测与价值判断解耦
- **层**：L3 Pattern ｜ **aggregation_rule**: **R1 机制簇**（EK-03/EK-04/EK-07/EK-09，共享"polarity-free evaluator + 语义解析"机制，跨 evaluator + result 子系统）
- **簇内 EK**：EK-03(polarity-free) ↔EK-04(三值组合) ↔EK-07(probe 反转) ↔EK-09(bool=result.safe)
- **一句话**：把"X 发生了吗"（检测）与"X 是好事还是坏事"（价值）分成两层——同一检测器可复用于攻击（DETECTED=坏）与探针（DETECTED=好）。
- **证据强度**：S4（test_xpia / test_probes 锁定同 evaluator 双语义；文档明示）
- **scope**：
  - applies_when：验证系统同时测"不该发生的"和"应该发生的"
  - does_not_apply_when：单一语义判定系统；引入反转会增加心智负担
- **epistemic**：Validated Pattern（RAMPART 内 S4；cross-project pending）

## KO-03 · 可观察性边界决定结论可信度（observability 不是可选项）
- **层**：L4 Cognitive Model ｜ **aggregation_rule**: **R2 因果链簇**（EK-01→EK-05→EK-09 因果链：声明观察边界→边界缺失时降级→影响最终判定）
- **簇内 EK**：EK-01(声明) →EK-05(降级) →EK-09(判定)
- **一句话**：验证 agent 的结论可信度 **上界** 由观测通道的完整性决定；缺失的观测必须降级为不确定，否则就是假装安全。
- **解释范围扩大**：适用于所有"基于不完整遥测下安全结论"的系统（日志缺失时的审计结论、无 tool 观测时的安全断言）。
- **证据强度**：S4（observability 降级 5 个单测）
- **scope**：
  - applies_when：判定依赖对被测系统的运行时观测
  - does_not_apply_when：被测行为完全确定（如纯文本生成的单一分类）
- **epistemic**：Cognitive Model（RAMPART validated；cross-project pending）

## KO-04 · 执行骨架 + 基础设施韧性：失败不拖垮验证套件
- **层**：L3 Pattern ｜ **aggregation_rule**: **R1 机制簇**（EK-08/EK-10/EK-12/EK-13/EK-26，共享"生命周期骨架 + 异常→ERROR"机制，跨 core/execution + attacks + probes）
- **簇内 EK**：EK-10(Template Method) ↔EK-08(异常→ERROR) ↔EK-12/13(两策略) ↔EK-26(早停)
- **一句话**：把验证的执行生命周期固化成骨架（固定事件顺序 + 跨切面关注），策略只写核心逻辑；任何基础设施异常降级为 ERROR 而非崩溃套件。
- **证据强度**：S4（test_execution 生命周期/broken handler 测试）
- **scope**：applies_when：验证框架需支持多执行策略且对抗环境不稳定（网络/API 波动）
- **epistemic**：Validated Pattern（cross-project pending）

## KO-05 · 安全工具的攻击者输入纵深防御（自噬防御）
- **层**：L4 Cognitive Model ｜ **aggregation_rule**: **R4 主题簇**（EK-17/EK-18/EK-20/EK-21/EK-23 围绕"框架自身必须防御它测试的输入"主题，覆盖终端/xdist/存储/judge 四通道）
- **簇内 EK**：EK-17(终端) + EK-18(xdist trust) + EK-20(大小限) + EK-21(存储路径) + EK-23(judge 输出)
- **一句话**：**一个安全测试框架必须把自己的攻击面当作被测对象**——它摄入的输入（agent 响应、payload、LLM judge 输出）都被攻击者控制过，因此每条通道都要有纵深防御。
- **解释范围扩大**：任何"测试对抗内容"的工具（fuzzer、pentest 工具、red team 平台）都适用。
- **证据强度**：S4（test_payload_store_security + xdist trust 测试 + terminal 测试）
- **scope**：
  - applies_when：工具会渲染/存储/传输/解析可能被攻击者控制的内容
  - does_not_apply_when：输入来自可信内部通道且无渲染
- **epistemic**：Cognitive Model（RAMPART 内强证据；cross-project pending）

## KO-06 · 概率行为的统计验证契约（trial gate）
- **层**：L4 Cognitive Model ｜ **aggregation_rule**: **R2 因果链簇**（EK-16→EK-19→EK-20 因果链：trial 克隆→聚合 gate→incomplete 强制失败）
- **簇内 EK**：EK-16(trial 克隆+gate) →EK-19(incomplete→FAIL) →EK-20(大小限→incomplete)
- **一句话**：对概率行为（LLM），单次通过不是证据；**验证契约是"阈值 × 完整执行"**——通过率必须达到阈值，且"跑完"本身是必要条件。
- **证据强度**：S4（test_plugin trial gate + incomplete exit 测试）
- **scope**：applies_when：被测行为概率性（LLM/随机策略）；does_not_apply_when：确定性行为（单测）
- **epistemic**：Cognitive Model（RAMPART validated；cross-project pending）

## KO-07 · 资源清理不变量：失败路径也必须保证释放
- **层**：L3 Pattern ｜ **aggregation_rule**: **R3 不变量簇**（EK-14 的 gather 防孤立 + EK-12 AsyncExitStack + EK-08 异常降级汇聚到同一不变量："任何执行路径后资源都被释放"）
- **簇内 EK**：EK-14(gather 防孤立) + EK-12(AsyncExitStack) + EK-08(异常→ERROR)
- **一句话**：并发资源管理的关键决策是"**失败的 sibling 不得阻止成功 sibling 注册清理**"——用非取消并发（gather+return_exceptions）而非会取消的 TaskGroup。
- **证据强度**：S4（test_partial_activation_failure_still_cleans_up_siblings）
- **scope**：applies_when：并发创建需外部副作用（远程 payload 上传）的资源；does_not_apply_when：无外部副作用（纯内存）
- **epistemic**：Validated Pattern（设计注释 + 测试锁定；cross-project pending）

## KO-08 · LLM 参与判定时的防自欺机制（重依赖隔离 + 输出强校验）
- **层**：L3 Pattern ｜ **aggregation_rule**: **R4 主题簇**（EK-22/EK-23/EK-24 围绕"LLM/重依赖进判定链路时的防护"主题）
- **簇内 EK**：EK-22(懒加载) + EK-23(judge 输出校验) + EK-24(桥隔离)
- **一句话**：把 LLM/重依赖接入判定链路时，三件事缺一不可：**隔离**（懒加载防启动污染）、**校验**（结构化输出强校验，非枚举值拒绝）、**降级**（LLM 失败→UNDETERMINED 而非猜测）。
- **证据强度**：S4（test_public_api 懒加载 + llm_judge 校验测试）
- **scope**：applies_when：判定链含 LLM 裁判或重上游依赖
- **epistemic**：Validated Pattern（cross-project pending）

---

## 数量与配比
- EK: 26（宽底座）→ KO: 8（窄尖顶，L3×4 + L4×4）
- 全部 KO 可回溯 EK（aggregation_rule 内列明）；无孤立 EK；无"同子系统=聚合"假聚合
- 无 L5（方法论）：单项目证据不足以升 L5；所有跨项目声明标 pending
