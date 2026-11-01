# 06 Validation & Evidence · RAMPART

## 6.1 验证方式（Blind Reconstruction 程序）
1. Truth Auditor：对每条 EK 引用仓库实际内容重新核验（file:line / test 名）
2. Coverage Auditor：独立重搜 rampart/ 全部 132 个 .py 的模块覆盖（见 6.6）
3. Flow Auditor：对 28 条 Flow Edge 逐一回溯 symbol/file/condition
4. Abstraction Auditor：独立判定 8 个 KO 的 L3/L4/L5 是否过度升维
5. Counterexample Hunter：对每个 L3+ KO 定向反例攻击（记录于 03 各 KO）
6. Epistemic Auditor：核查 Fact/Observation/Hypothesis/Pattern/Model/Principle 标注诚实性

## 6.2 判定统计
| 判定 | 数量 | 说明 |
|---|---|---|
| CONFIRMED | 22 | EK-01/02/03/04/05/06/07/08/09/10/11/12/13/14/15/16/17/24/25/26/28/30 等（代码+测试双证） |
| PARTIALLY_CONFIRMED | 6 | EK-18/19/20/21/23/27（机制确认，参数边界未完全验证） |
| OVER_GENERALIZED | 1 | 原 KO-02 草案曾称"所有 CI 门都应强制不完整失败"——超出 RAMPART 证据，已收紧为安全测试工具语境 |
| MISSING | 1 | 原 02 草案缺 converter（docx 转换）EK——补齐为 EK-37 证据链一部分（PyRIT WordDocConverter 映射） |
| CONTRADICTED | 0 | — |
| NEEDS_HUMAN_REVIEW | 2 | C-07（RAMPART vs 直接 PyRIT 脚本的价值边界）、C-09（pyrit pin 代价） |

## 6.3 盲重建对比（3 成功 + 3 错误示例）
**3 成功（盲重建命中原结论）**：
1. **判定优先级**：独立从 result.py 推导 attack DETECTED>UNDETERMINED>NOT_DETECTED / probe 反向 —— 与原 EK-04 一致
2. **观测降级**：独立从 _xpia.py:247-284 推导 SAFE+RESPONSE_ONLY+零工具→UNDETERMINED —— 与原 EK-03 一致
3. **trust boundary**：独立从 _xdist.py:1-50 推导 worker payload 不受信 + schema/enum/depth 校验 + ANSI 剥离 —— 与原 EK-24 一致

**3 错误（盲重建修正）**：
1. **最初假设 "UNDETERMINED 是全局保守降级"** → 实际是"组合短路只在决定性结果发生，UNDETERMINED 传播有精确规则"（_AllEvaluator 左 UNDETERMINED 不短路，防顺序依赖）——修正为 EK-17 的精确语义
2. **最初假设 "LLM judge 失败一律 UNDETERMINED"** → 实际分两类：解析/空响应/限流→UNDETERMINED（降级），配置/认证→EvaluatorError→基础设施 ERROR（传播）——修正为 EK-19 的两类语义
3. **最初假设 "gather(return_exceptions=True) 仅为容错"** → 实际核心动机是清理注册顺序（TaskGroup 取消会孤儿化远端 payload）——修正为 EK-33 的清理保证动机

## 6.4 反例压力（Counterexample 摘要）
| KO | 反例攻击 | 结果 |
|---|---|---|
| KO-01 | 若观测声明宽松（RESPONSE_ONLY 判 SAFE 不降级）→ 假阴性 | 测试 test_xpia.py:295 明确禁止 → 结论成立 |
| KO-01 | 若要求全通道才 SAFE → 假阳性泛滥 | RAMPART 取中间（声明即取信 + 弱观测降级）→ 边界成立 |
| KO-02 | 若 503 计入攻击成功率 → 统计污染 | sink.py:196 显式剔除 → 结论成立 |
| KO-02 | 若 incomplete 不强制失败 → 丢分片全绿通过 | plugin.py:456 强制 → 结论成立 |
| KO-03 | 若只在传输层消毒 → terminal/judge 注入 | 四处独立防御（渲染/存储/评估/传输）→ 结论成立 |
| KO-04 | 若 AND 在 UNDETERMINED 左短路 → A&B≠B&A | 设计注释显式反例 + 测试覆盖 → 结论成立 |
| KO-06 | 若用 TaskGroup 并发 handle entry → 孤儿远端 payload | 注释 + test_xpia.py:231 → 结论成立 |
| KO-08 | 若 LLM 失败降级 NOT_DETECTED → 系统假阴性 | llm_judge.py:496 显式降级 UNDETERMINED → 结论成立 |

## 6.5 Epistemic 审计
- KO-01~08 全部标注 Validated Pattern（RAMPART 内多机制/测试印证），无一冒充 Principle/Law
- CK-03 显式标 cross-project pending（仅 RAMPART 单一证据）
- C-01~C-09 全部保持 Hypothesis/暂定/待复核标注，未写入 03 层

## 6.6 Coverage 审计（模块覆盖）
- rampart/ 包 132 .py：直接精读 38 个核心文件（core 全部 17 + pytest_plugin 全部 8 + evaluators 4 + attacks 2 + probes 2 + payloads 4 + drivers 3 + pyrit_bridge 1 + reporting 2 + surfaces 1 + common 2 + converters 1 + skills 1）
- 覆盖决策相关模块 100%；工具/脚本（scripts/hatch_build.py 等 CI 辅助）经 pyproject 与 CI 配置侧面覆盖
- 测试揭示行为：从 8 个测试文件（test_xpia/test_execution/test_trace/test_llm_judge/test_xdist/test_xdist_aggregation/test_plugin/test_payload_store_security）提取行为证据
- **反证搜索**：主动检索"是否有什么机制推翻 SAFE 判定"——找到 xdist worker 崩溃保留已流式结果（test_xdist_aggregation.py:406）、teardown-only 结果不流式（:333）等边界行为，均已纳入 EK-41

## 6.7 Reconciliation 记录
- 独立验证修正已并入：EK-17 短路语义精确化、EK-19 两类失败语义、EK-33 清理注册动机
- 原始考古产物不做任何修改（验证阶段独立）；修正体现在本 Package 终稿
- 无 CONTRADICTED 项；无跨层冲突（Flow 交叉校验全通过）

## 6.8 质量指标
| 指标 | 值 |
|---|---|
| EK 数（含 links） | 35 |
| 游离 EK | 0（35/35 有 links） |
| EK 平均出边 | ≥1（满足） |
| KO 数（含 aggregation_rule） | 8 |
| 聚合规则覆盖率 | 100% |
| KO 簇规模范围 | 3-6（满足 3~12） |
| Candidates | 9（4 跨项目 + 2 暂定 + 3 待复核） |
| Flow Edge | 28（七类流，可回溯） |
| CONFIRMED / PARTIALLY / OVER-GEN / MISSING / CONTRADICTED / NEEDS_REVIEW | 22 / 6 / 1 / 1 / 0 / 2 |
| 反例覆盖 | 8/8 KO 有定向反例攻击 |
| 证据强度分布 | S3（实现）+ S4（测试验证）为主；S8 无（跨项目验证待做，已标 pending） |

## 6.9 仍存在的缺口
- **Runtime 验证缺失**：本考古为只读代码考古，未实际运行 pytest（RAMPART 测试需要 PyRIT/LLM 端点与网络）；S5+ 证据待运行时验证
- **xlst 分布式真机验证**：xdist 行为证据来自代码+集成测试文件，未在真实多机环境跑
- **LLM judge 端到端**：integration 测试依赖 .env（LLM 端点），未在本次考古执行
