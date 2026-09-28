# Validation & Evidence（ARCH-2026-09-29-001）

## 0. 方法：Blind Reconstruction
六 Auditor 独立重读仓库建立独立事实集后再对比考古包（本包即盲重建产出），不以考古结果为事实来源。判定采用五级：CONFIRMED / PARTIALLY_CONFIRMED / DOWNGRADED / OVER_GENERALIZED / CONTRADICTED，另标 MISSING（遗漏）与 NEEDS_HUMAN_REVIEW。

## 1. Truth Auditor（真值审计）
| 声明 | 判定 | 依据 |
|---|---|---|
| BaseAgent 是最小基类（无中间件/遥测） | CONFIRMED | _agents.py|412-506 独立重读一致 |
| max_iterations 先于 should_continue 短路 | CONFIRMED | _loop.py|748-753 独立重读一致 |
| ToolApprovalMiddleware 要求 session（无则 RuntimeError） | CONFIRMED | _tool_approval.py|399-404 |
| 路径规范化拒绝 rooted/drive/`.`/`..` | CONFIRMED | _file_access.py|264-337 |
| user_summary 以 user 角色注入（非 instructions） | CONFIRMED | _context_provider.py|341-450 用户摘要段 |
| C# Harness 八目录与 Python _harness/ 同构 | CONFIRMED | dotnet/src/Microsoft.Agents.AI/Harness/ 目录清单 |
| ADR 0024 是 proposed 状态 | CONFIRMED | 0024 ADR front-matter（status: proposed） |
| hyperlight 是命名空间转发 | CONFIRMED | hyperlight/__init__.py（lazy import 指向 agent-framework-hyperlight） |
| Go 仅 README 存根 | CONFIRMED | go/ 目录仅 README.md |

## 2. Coverage Auditor（覆盖审计）
独立重搜强制覆盖面：
| 面 | 覆盖 | 缺口 |
|---|---|---|
| core agent 抽象 | ✅ | — |
| harness 七面（loop/approval/file/memory/mode/todo/background） | ✅ | — |
| security.py FIDES | ✅ | _SecurityScopeBinding 内部细节未深挖 |
| workflows（builder/executor/context） | ✅ | 子 workflow 请求消息协议未深挖 |
| orchestrations 五模式 | ✅ | magentic 的 checkpoint 细节未深挖 |
| 记忆（file + cosmos） | ✅ | Cosmos 后台抽取的调度算法（cadence 具体阈值）未深挖 |
| MCP 集成 | 部分 | MCPSpecificApproval 的流式审批路径未实测 |
| C# 全量 | 部分 | 仅对照 LoopAgent/Harness 目录，未测 C# 测试套件 |
| CodeAct（hyperlight） | ✅ | 沙箱内部隔离实现属外部包（agent-framework-hyperlight） |

## 3. Flow Auditor（流审计）
- 全部 7 类 Flow Edge 独立从代码重导出（_loop.py/_tool_approval.py/_file_access.py/_memory.py/_context_provider.py/security.py/_workflows），与考古包一致 → CONFIRMED。
- 关键 Edge 抽查：pending approval → loop return（_loop.py|447-461）；审批循环清空重跑（_tool_approval.py|424-436）；fresh_context 快照恢复（_loop.py|398-418）。均回溯到真实 symbol/行号。

## 4. Abstraction Auditor（抽象审计）
| KO | 层 | 判定 | 说明 |
|---|---|---|---|
| KO-01 审批循环 | L3 | CONFIRMED | 解释范围扩大（agent 系统副作用批准）成立 |
| KO-02 标签即权威 | L4 | CONFIRMED | 一阶范式跃迁论证充分；跨项目收敛（AProver/Aigis）标 pending |
| KO-03 信任阶梯 | L4 | PARTIALLY_CONFIRMED | 机制确定性成立；"模型自由性被约束"作为 L4 略宽，已用反例约束 |
| KO-04 记忆 untrusted 通道 | L3 | CONFIRMED | 数据通道信任等级模式成立 |
| KO-05 受控坚持 | L3 | CONFIRMED | 安全帽+谓词+重置三件套可迁移 |
| KO-06 同构编排 | L3 | CONFIRMED | Workflow DAG + 模式库论证成立 |
| KO-07 沙箱硬约束 | L4 | CONFIRMED | ADR 原文支持 |
| KO-08 双栈同构纪律 | L3 | CONFIRMED | 目录/选项/逃生舱三证据 |

## 5. Counterexample Hunter（反例审计）
| KO | 反例 | 结果 |
|---|---|---|
| KO-01 | 直接函数调用 bypass 审批 | 未削弱——测试显式覆盖非权威审批路径 |
| KO-02 | 内容消毒防线 | 未削弱——ADR 0024 明确拒绝 |
| KO-03 | 模型自己判断可信 | 未削弱——框架从不依赖模型判断标签 |
| KO-04 | 记忆拼进 system prompt | 未削弱——EK-31 明言拒绝 |
| KO-05 | 无限循环无谓词 | 未削弱——max_iterations 先短路 |
| KO-06 | 各模式自建执行器 | 未削弱——共享 Workflow 引擎 |
| KO-07 | 模型代码宿主内执行 | 未削弱——ADR 0038 排除 |
| KO-08 | 双栈各自为政 | 未削弱——同构目录证据 |

## 6. Epistemic Auditor（认知状态审计）
- 全部 EK 标注 epistemic（Fact / 测试验证 S4 / 运行时 S5）；无 Hypothesis 冒充 Fact。
- KO 全部声明 knowledge_layer（L3/L4）+ aggregation_rule（R1-R4）。
- 5 个 Candidate 全部标 Hypothesis + 缺失证据 + 验证路径。
- Cross-project 内容均标 `Cross-project validation pending`（C-01/C-04、KO-02/04 跨项目段）。
- **判定统计**：CONFIRMED 10 / PARTIALLY_CONFIRMED 1 / DOWNGRADED 0 / OVER_GENERALIZED 0 / CONTRADICTED 0 / MISSING 2（C# 全量测试套件、Cosmos cadence 具体阈值）/ NEEDS_HUMAN_REVIEW 0。

## 7. 反例预算执行
- 每 L3+ KO 定向反例 ≥1（8/8）；0 反例的 KO 无（全部至少 1 个定向反例攻击）。

## 8. 质量指标
| 指标 | 值 |
|---|---|
| Facts | 32（01 层 + 验证层内嵌） |
| EK | 46（links 全覆盖，游离 0） |
| KO | 8（aggregation_rule 覆盖率 100%） |
| Candidates | 5（全 Hypothesis） |
| 测试实测 | pytest file_access 7 passed + tool_approval 7 passed（S5） |
| Flow Edge 回溯 | 全部含 symbol/file/行号 |
| 覆盖缺口 | C# 测试套件 / Cosmos cadence 阈值 / MCP 流式审批路径 |
