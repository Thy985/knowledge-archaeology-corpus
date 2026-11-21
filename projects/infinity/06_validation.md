# Validation & Evidence — Infinity

## 六审计结果
| 审计 | 结果 | 要点 |
|---|---|---|
| Truth | **PASS** | 45+ 采证锚全部 file:line 可回溯；无 S0-S1 级结论冒充事实 |
| Coverage | **PARTIAL** | 核心 runtime + lambda + rap + sandbox 覆盖；daemon session 深层状态机/UI/Bedrock provider 细节未覆盖（已声明） |
| Flow | **PASS** | 34 Edge 全可回溯 symbol/file/condition；Flow→KO 交叉校验 8/8 |
| Abstraction | **PASS（4 pending）** | 8 KO 均经 promotion 论证；KO-D01/D02/D05/D08 标 Cross-project validation pending |
| Counterexample | **PASS** | 反例预算：KO-D05 反例（真实工具/长连接破坏纯数据假设——RAP 存在即证明）；KO-D03 反例（NoPendingCall 静默忽略可能掩盖丢失）；无 0 反例项 |
| Epistemic | **PASS** | Fact/Observation/Hypothesis/Pattern/Model/Principle 严格区分；性能基准标来源（博客 S5）非仓库事实 |

## 质量指标
| 指标 | 值 |
|---|---|
| EK | 24 条（EK Graph，平均出边 2.75，游离 0） |
| KO | 8 个（R1×2/R2×2/R3×1/R4×3；L3×2/L4×5/L5×1） |
| Flow | 34 Edge（Control 5/State 5/Data 5/Evidence 4/Authority 5/Memory 5/Policy 5） |
| Candidates | 7（跨项目 3 / 暂定 3 / 已排除 1） |
| 聚合规则覆盖率 | 100%（8/8 KO 声明 aggregation_rule） |
| 测试/代码比 | 0.24x（11,782/48,484） |

## 待 IVR 攻击面（阶段⑤）
1. 计数类断言：EK 平均出边 2.75 / Flow 34 / 系统表（无——lambda 存储非表模型）/ KO 层级分布
2. 过度升维风险：KO-D05（L5）"agent 即数据"是否基于单一项目→必须 pending；KO-D02 commit barrier 是否与 dbos 混淆
3. Flow Edge 真实性：EF-3（drop 用户输入）与 D08 的边界；PF-4 失败恢复是否真的防悬挂
4. bypass/alternate 路径：NoDeferral 是否绕过 defer 策略；drop_unvalidated 是否绕过幂等；同步工具是否绕过 dispatch barrier
5. Epistemic：性能基准标注；compaction 阈值（C-D06）是否误写为事实

## Benchmark case（供 skill CI）
**Case**: "serverless agent 在等待工具结果时如何做到零计算占用？"
- 期望：agent 状态持久化 → 进程退出 → 事件到达 → 从持久化状态恢复
- Infinity 证据：slice 语义（thread.rs:106-118）+ SQS 单 delivery 单 slice（event_handler.rs:174-178）+ NoDeferral（defer.rs）+ DynamoDB 幂等（state_store.rs:48-80）
- 反例面：pending 工具调用期间到达的 synthetic 事件 → NoDeferral 中断（接受语义）vs resident defer；睡眠 → EventBridge timer（sleep.rs）
- 对照：restate（journal 重放）/ dbos（数据库 checkpoint）——三种"等待期间零占用"的实现路径
