# 05 Candidates · RAMPART（未验证假说 / 跨项目候选）

## 5.1 跨项目假说（Cross-project Candidates）
| ID | 假说 | 层级 | 当前证据 | 缺失证据 | 验证路径 |
|---|---|---|---|---|---|
| C-01 | "观测声明契约"（adapter 声明可观测通道 → 判定区分 UNDETERMINED/NOT_DETECTED）是可迁移的安全测试框架设计模式 | L3 Pattern 假设 | RAMPART 内多机制印证（KO-01） | 仅单一项目；dsh-pentest/silver-shield 无等价声明机制 | 考古其他 agent 安全/eval 框架时检查是否有 observability declaration |
| C-02 | "被测 agent 输出当不受信输入"（纵深防御四处独立消毒）是 agent 测试工具通用要求 | L3 Pattern 假设 | RAMPART 强证据（KO-03） | 未与 agentevals/opencode corpus 交叉 | 在 agentevals/opencode 产物中检索输入消毒处理 |
| C-03 | LLM-as-judge 四层可靠性清单（JSON 契约/失败降级/加固/可复现）可迁移到任何 LLM 判定场景 | L3 Pattern 假设 | RAMPART 强证据（KO-08） | 未跨项目验证；LLM judge 在评测工具中常见但工程化程度差异大 | 对照 agentevals/dart-agent-core 的 judge 设计 |
| C-04 | 不完整运行强制失败（green must mean ran and passed）对 CI 安全门普遍成立 | L5 方法论假设 | RAMPART 明确（KO-02） | 未对照其他 CI 工具（coverage/fuzz gate）实践 | 检索 corpus 中 CI 门控相关项目 |

## 5.2 暂定模式（未确认）
| ID | 暂定模式 | 说明 |
|---|---|---|
| C-05 | "任务组 vs gather(return_exceptions)"资源清理取舍：并发资源注册场景下 TaskGroup 取消语义会孤儿化远端资源 | RAMPART 注释与测试支持（EK-33），但属单一代码库观察；是否普遍成立需跨项目确认 |
| C-06 | "三值布尔组合短路规则"（OR 短路真/AND 短路假，UNDETERMINED 不短路）为通用三值逻辑约束 | 逻辑上自洽（防顺序依赖），但 RAMPART 是唯一实现样本 |

## 5.3 不确定结论（需人类复核）
| ID | 不确定点 | 原因 |
|---|---|---|
| C-07 | RAMPART 是否系统性优于"直接写 PyRIT 脚本做红队"（不做 pytest 框架时）——RAMPART 的增量价值边界 | 需实际使用对比；代码只证明设计意图（EK-38），未证明效率/覆盖优势 |
| C-08 | xdist trust boundary 的 MAX_METADATA_DEPTH=6 与 16MB 上限是否覆盖真实对抗场景 | 参数来自代码常量（_xdist.py:50-60），未见压力测试证据 |
| C-09 | pyrit==0.13.0 精确 pin 的动机（生态控制 vs 兼容性代价）——scripts/bump_pyrit_version.sh 存在说明版本升级是受控流程 | 有脚本证据（bump_pyrit_version.sh），但升级频率/代价无数据 |
| C-10 | NEUTRAL_EVALUATOR 的 \"ambiguous→NOT_DETECTED\" persona 指引是否被用户误读为\"模糊即安全\"（框架层实际以 UNDETERMINED 处理调用失败）——文档/教程层面的认知风险 | personas.py 与 llm_judge.py 语义分层证据齐全，但用户认知影响无数据 |

## 5.4 跨项目连接记录
- **PyRIT（corpus ARCH-2026-10-04）**: RAMPART 构建于 PyRIT 之上（EK-36/37/38），是"生态之上"的直接对照样本——RAMPART 证明 PyRIT 可被封装为更高层安全语义
- **dsh-pentest / silver-shield / E2E-CLI**: 用户项目族（Agent 安全方向），C-01/C-02 的候选交叉验证对象
- **agentevals（corpus ARCH-2026-09-12）**: eval 框架，C-03 的对照对象（LLM judge 工程化对比）
