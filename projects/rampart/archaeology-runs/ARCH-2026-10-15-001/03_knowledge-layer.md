# 03 Knowledge Layer（Generalized KO）· RAMPART

8 个 Core KO，每个声明 aggregation_rule（R1-R4）+ 簇内 EK + 解释范围扩大论证 + 反例压力。

## KO-01 观测通道即判定契约：不可观测 ≠ 不安全（R3 不变量簇）
- **aggregation_rule**: R3 —— {EK-01, EK-02, EK-03, EK-04, EK-05} 经 constraint 边汇聚到同一安全不变量：判定必须区分"没发生"与"看不到"
- **L4 认知模型**: 安全评估系统的判定诚实性取决于观测声明的精度——声明可观测通道即取信其内容，缺失通道则输出"不知道"（UNDETERMINED）而非"没发生"（NOT_DETECTED）；当"安全"结论恰好出现在观测最弱的条件下（RESPONSE_ONLY + 零工具观测），结论本身被降级
- **解释范围扩大**: 不止适用于 RAMPART——任何基于 trace/telemetry 的 agent 安全判定（eval harness、LLM gate、审计系统）都面对同一陷阱：把"观测不到"当作"未发生"
- **反例压力**: ①若观测通道声明宽松（RESPONSE_ONLY 也算 SAFE），套件会系统性假阴性（test_xpia.py:295 明确禁止）；②若要求全通道才判 SAFE，套件会系统性假阳性（UNDETERMINED 泛滥）——设计取中间：声明即取信 + 弱观测时降级
- **Epistemic**: Validated Pattern（RAMPART + dsh-pentest/silver-shield 候选交叉验证 pending）

## KO-02 故障与发现分离：基础设施错误不是安全发现（R3 不变量簇）
- **aggregation_rule**: R3 —— {EK-06, EK-07, EK-39} 汇聚到不变量：ERROR ≠ UNSAFE ≠ UNDETERMINED，且"测试运行不完整"本身是失败
- **L4 认知模型**: 安全测试工具的价值在于它的绿灯有意义——"green 必须意味着 ran and passed"；把基础设施故障混入安全统计（分母/分子）会同时制造假阳性和假阴性
- **解释范围扩大**: CI 安全门（coverage gate、fuzz gate、dependency gate）的通用教训：不完整运行必须显式失败，不能因"存活测试全绿"而通过
- **反例压力**: 若把 503 当 UNSAFE，攻击成功率被基础设施波动污染（sink.py:196 明确剔除）；若把 503 当 SAFE，则掩盖服务故障
- **Epistemic**: Validated Pattern（RAMPART 内多机制印证：ERROR 转换 + 分母剔除 + incomplete 强制失败）

## KO-03 信任边界纵深防御：被测系统输出 = 不受信输入（R1 机制簇）
- **aggregation_rule**: R1 —— {EK-24, EK-25, EK-26, EK-27, EK-28, EK-20} 共享消毒机制且跨 ≥2 子系统（xdist 传输 / 报告渲染 / payload 存储 / LLM judge prompt）出现
- **L3 Pattern**: 当系统要处理"可能被对抗性控制的文本"（agent 输出、注入 payload、LLM 响应）时，在每个信任边界都做独立消毒：传输层 JSON-safe + schema/enum/depth 校验、渲染层 ANSI 剥离 + safe_str、存储层路径包含、评估层 prompt 加固
- **解释范围扩大**: 这是"处理不受信数据"通用模式在 AI 测试工具中的实例（同类：HTML sanitizer、日志注入防护、prompt injection 防御）
- **反例压力**: 单点消毒不足——RAMPART 在 xdist 边界、terminal 渲染、report sink、judge prompt 四处独立防御（若只在传输层消毒，agent 输出的 ANSI 会在 terminal 注入）；"每个边界独立防御"是本簇结论，非单一机制
- **Epistemic**: Validated Pattern（RAMPART 内跨子系统印证）

## KO-04 组合评估器必须保留"不知道"：短路与 UNDETERMINED 传播（R1 机制簇）
- **aggregation_rule**: R1 —— {EK-17, EK-05, EK-14} 共享组合传播机制，跨评估器组合与在线停止出现
- **L4 认知模型**: 布尔组合（OR/AND/NOT）作用于三值逻辑（DETECTED/NOT_DETECTED/UNDETERMINED）时，短路优化必须只发生在"决定性结果"上——OR 短路于真、AND 短路于假；"不知道"不能短路，否则结果依赖操作数书写顺序
- **解释范围扩大**: 任何三值逻辑组合（安全门、审批流、检测链）的通用设计约束；RAMPART 的 `_AllEvaluator` 注释显式给出反例："returning early there would make the outcome depend on the order the operands were written in"
- **反例压力**: 若 AND 在 UNDETERMINED 左操作数短路，A&B vs B&A 会得到不同判定——RAMPART 测试明确覆盖此顺序无关性
- **Epistemic**: Validated Pattern

## KO-05 生命周期骨架 + 事件系统：框架横切关注点与套件弹性（R2 因果链簇）
- **aggregation_rule**: R2 —— {EK-08, EK-09, EK-10, EK-11, EK-12} 沿 causal 链（骨架→事件工厂→handler→trial）
- **L3 Pattern**: 测试框架把"执行骨架"固定为基类（pre/post/error 三事件 + 异常统一转结果对象），子类只实现策略核心；事件 handler 可插拔且**handler 失败不得影响测试结果**——"单测试失败不拖垮套件，handler 失败不压制结果"
- **解释范围扩大**: pytest plugin 架构 / CI 工具设计通用模式（同类：pytest hooks、JUnit listeners）
- **反例压力**: handler 若失败即中止，一个坏的 observability handler 会废掉整个测试运行（RAMPART 选择 log+swallow 保结果）；trial 若共享 execution 实例，状态会跨 trial 泄漏（每 trial 新 factory 是显式设计）
- **Epistemic**: Validated Pattern

## KO-06 适配器协议与注入生命周期：依赖反转让策略与实现解耦（R2 因果链簇）
- **aggregation_rule**: R2 —— {EK-30, EK-31, EK-32, EK-33, EK-34} 沿 causal 链（Adapter 三要素→InjectionHandle 协议→surface→XPIA 七阶段）
- **L3 Pattern**: 注入类攻击的执行策略只依赖"进入/退出上下文管理器"的抽象（InjectionHandle），从不依赖具体数据源实现；并发激活用 gather(return_exceptions=True) 而非 TaskGroup——避免取消导致"已创建未注册清理"的远端资源孤儿化
- **解释范围扩大**: 资源生命周期管理的通用教训：清理注册必须先于错误传播；测试框架的通用模式：策略层对集成点只依赖协议
- **反例压力**: TaskGroup 取消语义（首错即取消）会跳过已创建 handle 的 __aexit__ 注册——远端已上传 payload 成孤儿（test_xpia.py:231 覆盖）；若 strategy import Surface 具体类，新增数据源需改策略代码
- **Epistemic**: Validated Pattern

## KO-07 PyRIT 生态边界：复用底层原语但不泄漏类型（R4 主题簇）
- **aggregation_rule**: R4 —— {EK-36, EK-37, EK-38} 围绕"RAMPART×PyRIT 关系"覆盖互补维度（边界/映射/增量）
- **L3 Pattern**: 构建于成熟底层库之上的框架应"内部复用、外部自持"——公共 API 暴露自己的类型（LLMConfig/Persona/Payload），把底层库类型隔离在 bridge 层；依赖精确 pin（pyrit==0.13.0）控制生态漂移
- **解释范围扩大**: SDK/框架封装通用模式（同类：LangChain 之上的安全层、requests 之上的 HTTP 封装）；与 corpus 已考古的 PyRIT（ARCH-2026-10-04）构成直接生态对照
- **反例压力**: 若公共 API 泄漏 PyRIT 类型，消费者被迫同时跟进两个生态的 breaking changes（精确 pin 缓解）；若完全不依赖底层，则重复造轮子（RAMPART 明言不薄封装：安全语义/编排/pytest 集成是增量）
- **Epistemic**: Validated Pattern（corpus 内 PyRIT 交叉印证）

## KO-08 LLM-as-judge 可靠性工程：JSON 契约、失败降级、加固、可复现（R4 主题簇）
- **aggregation_rule**: R4 —— {EK-18, EK-19, EK-21, EK-23} 围绕 LLM judge 可靠性覆盖互补维度（契约/降级/scope/可复现）
- **L3 Pattern**: 用 LLM 做判定器的系统需要四层保障：①严格输出契约（JSON schema + 字段校验）；②失败降级语义（解析/空响应/限流 → "不知道"而非"没发生"）；③输入加固（transcript 标记为数据而非命令）；④可复现配置（temperature=0 + seed）
- **解释范围扩大**: 任何 LLM-as-judge / LLM-as-filter 系统的通用工程清单（评测、内容审核、自动打分）
- **反例压力**: 若 LLM 失败降级为 NOT_DETECTED，系统会系统性假阴性（RAMPART 显式禁止）；若无加固后缀，transcript 里的注入指令可劫持 judge
- **Epistemic**: Validated Pattern

## Core Knowledge（跨对象归纳）
- **CK-01（L5 方法论）**: 设计 agent 安全测试系统时，先定义"观测声明"与"判定语义"的契约，再写任何执行逻辑——判定诚实性（UNDETERMINED 的存在）比判定灵敏度更重要，因为假阴性直接破坏工具的可信度
- **CK-02（L4）**: 安全测试工具的可信度 = 它的"没发现"的语义精度 + 它的"测试运行完整"的保证；两者缺一，绿灯无意义
- **CK-03（L4, cross-project pending）**: 被测 AI 系统的输出在工具内部必须被当作不受信输入处理（与处理用户输入同级的纵深防御）——这一认知来自 RAMPART，与 E2E-CLI/agentevals 的跨项目验证待做

## 三层配比检查
- Facts: 45+（01 Project Layer + snapshot）→ EK: 35 → KO: 8 → CK: 3
- EK 平均出边：全部 ≥1；游离 EK：0（35/35 有 links）
- 聚合规则覆盖率：100%（8/8 KO 声明 aggregation_rule）
- KO 簇规模：5/3/6/3/5/5/3/4（范围 3-6，符合 3~12）
