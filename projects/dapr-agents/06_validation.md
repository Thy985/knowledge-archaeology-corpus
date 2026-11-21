# Validation & Evidence — Dapr Agents

## 六审计结果
| 审计 | 结果 | 要点 |
|---|---|---|
| Truth | **PASS** | 全部 EK/KO/Flow 证据锚 file:line 可回溯；品类卡文档层事实（Agent deprecated）与代码事实（无 Agent 类）严格区分 |
| Coverage | **PARTIAL** | 核心（agents core / executor 审批 / workflow runner / 工具桥 / 流式 / 编排）覆盖；未覆盖：storage 三族实现细节 / document 模块 / observability wrappers / ext（drasi/claude）集成 |
| Flow | **PASS** | 28 Edge 可回溯；Flow→KO 交叉校验 7/7 |
| Abstraction | **PASS（3 pending）** | 7 KO 均经 promotion 论证；KO-D01/D03/D05 标 Cross-project validation pending |
| Counterexample | **PASS** | 反例预算：KO-D05 反例（长连接/外部副作用在重放模型下需特殊处理）；KO-D03 反例（logger 覆盖率 C-D06；hook 位置键控非确定性警告）；无 0 反例项 |
| Epistemic | **PASS** | 运行行为证据（中断 demo）未实跑（无 Docker），标"运行级 S5 待补"；不冒充已验证 |

## 质量指标
| 指标 | 值 |
|---|---|
| EK | 20 条（EK Graph，平均出边 2.35，游离 0） |
| KO | 7 个（R1×2/R2×2/R3×1/R4×2；L3×2/L4×4/L5×1） |
| Flow | 28 Edge（Control 4/State 5/Data 4/Evidence 3/Authority 5/Memory 4/Policy 3） |
| Candidates | 6（跨项目 3 / 暂定 3） |
| 聚合规则覆盖率 | 100%（7/7） |
| 测试/代码比 | 0.81x（41,490/51,516） |

## 待 IVR 攻击面（阶段⑤）
1. 计数类断言：EK 平均出边 2.35 / Flow 28 / EK 20 / KO 7
2. 过度升维风险：KO-D05（L5）"agent 即 workflow"是否与 infinity 混淆（两个实现端点）；KO-D01 workflow 骨架是否过度推广
3. Flow Edge 真实性：EF-1（重放日志抑制路径）、PF-2（至少一次 delivery）、AF-5（跨 app ToolError）
4. bypass/alternate 路径：subscribe 模式是否绕过 serve 的 HITL 路由；标准循环（无 executor）是否绕过审批；MCP 工具是否绕过 bridge 限制
5. Epistemic：中断 demo 未跑（S5 缺失）是否影响 KO-D02 层级

## Benchmark case（供 skill CI）
**Case**: "多 agent 系统中，进程崩溃后如何保证长时程 agent 工作流不从头重跑、不重复执行已完成 LLM/工具调用？"
- 期望：workflow 步骤持久化 → 崩溃 → 重启 → 从最后持久化步骤继续（重放不重执行）
- Dapr Agents 证据：executor-as-activity（durable_executor.py:36-58）+ 双 state store（configs.py:246）+ 确定性 id（agent_tool.py:198）+ HITL 审批确定性（durable_executor.py:141-218）
- 反例面：无 Docker 未实跑中断 demo（S5 待补）；工具副作用可重入性依赖工具实现（C-D03）
- 对照：infinity（slice 挂起恢复）/ restate（journal 重放）/ dbos（数据库 checkpoint）——四种"崩溃后续跑"实现路径
