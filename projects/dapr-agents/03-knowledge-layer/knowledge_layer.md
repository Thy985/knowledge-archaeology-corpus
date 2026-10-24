# Knowledge Layer — dapr-agents Generalized KO（L3/L4，6 条）

> 每个 KO 声明 aggregation_rule（R1-R4）+ 解释范围扩大论证；未跨项目验证标 `Cross-project validation pending`。

## KO-D01 · durable execution 三件套（L4 Cognitive Model · R1 机制簇）
- **aggregation_rule**: R1 机制簇——EK-D02（activity 幂等）+EK-D05（确定性 UUID）+EK-D19（崩溃恢复）共享"确定性/幂等"机制且跨 ≥2 子系统（LLM 调用/审批/启动）
- **簇内 EK**: D02/D05/D19
- **一句话稳定关系**: **可靠的长时间运行 agent 工作流 = 持久化点（entry state）+ 幂等活动（replay 缓存）+ 确定性 ID（uuid5）三件套——缺任一，崩溃恢复就会重放或重复**
- **解释范围扩大**: 从 Dapr workflow → 任何长时程 agent 系统（Temporal 式设计同线）
- **反例（预算内）**: 无 timeout 审批（None）挂起会阻塞——"可靠"不覆盖"可终结性"
- **epistemic**: Cognitive Model（MXC 的 fail-closed 链条与其互补；`Cross-project validation pending`）

## KO-D02 · 审批默认 fail-closed（L3 Pattern · R3 不变量簇）
- **aggregation_rule**: R3 不变量簇——EK-D05（超时/解析失败 auto-deny）+EK-D06（无流短路）+EK-D04（Deny 合成消息）汇聚到同一不变量："外部批准缺省即拒绝"
- **簇内 EK**: D04/D05/D06
- **模式**: 超时=拒绝、解析失败=拒绝、无交互通道=跳过（短路），只有显式配置无限等待才挂起——与 MXC telemetry 未知值 Blocked / strands-box DenyReason 同构
- **解释范围扩大**: HITL 通道的设计基线（跨项目证据：MXC/strands-box）
- **epistemic**: Pattern（跨项目对照成立）

## KO-D03 · workflow 与 LLM 的确定性边界（L3 Pattern · R2 因果链簇）
- **aggregation_rule**: R2 因果链簇——EK-D01（I/O activity 化）→EK-D02（重放检测）→EK-D09（消息链完整性）→EK-D10（mid-loop 省略 tools）：确定性主链
- **簇内 EK**: D01/D02/D09/D10
- **模式**: LLM 非确定性输出被 workflow 确定性外壳约束：I/O 全部 activity、重放返回缓存、消息链约束显式处理（skip_messages）、结构化输出窗口防模型"跑偏"
- **解释范围扩大**: 任何"LLM-in-workflow"系统的通用纪律
- **epistemic**: Pattern（dapr 内多子系统一致）

## KO-D04 · 扩展点双轨：决策型 hooks + 拦截型 lifecycle（L3 Pattern · R4 主题簇）
- **aggregation_rule**: R4 主题簇——EK-D04（hooks 五态）+EK-D15（lifecycle Protocol）覆盖"扩展机制"主题互补维度
- **簇内 EK**: D04/D15
- **模式**: 需要结果的策略（批准/改写/跳过）走 hooks 决策模型；需要过程拦截的横切（auth/OBO/观测）走 lifecycle Protocol——两种扩展点分离，框架核心保持 OSS-clean
- **解释范围扩大**: agent 框架策略注入的一般设计（对照 langgraph 回调）
- **epistemic**: Pattern

## KO-D05 · 状态分层：execution / session / tool-history（L4 Cognitive Model · R4 主题簇）
- **aggregation_rule**: R4 主题簇——EK-D12（session manifest+chunk）+EK-D17（双 store）+EK-D09（tool_history）覆盖"状态如何组织"互补维度
- **簇内 EK**: D09/D12/D17
- **一句话稳定关系**: **执行状态（workflow）、会话记忆（session store）、观测历史（tool_history）是三个生命周期和访问模式不同的状态层——分层存储比单一大状态更抗崩溃、更可审计**
- **解释范围扩大**: 与 letta-code（记忆分层）对照；Memory 周架构参照
- **epistemic**: Cognitive Model（`Cross-project validation pending`）

## KO-D06 · 显式扩展成本（L4 Cognitive Model · R1 机制簇）
- **aggregation_rule**: R1 机制簇——EK-D07（agent-as-tool child instance + depth）+EK-D14（MCP 声明式发现）+EK-D13（executor binding）共享"扩展必须走显式声明路径"机制
- **簇内 EK**: D07/D13/D14
- **一句话稳定关系**: **让"加一个能力"（新工具/新 agent/新执行模型）的接入成本显式化（child instance 隔离、声明式发现、binding 契约），比事后审计更省力**（与 MXC ProbeCoverage 强制声明同构）
- **解释范围扩大**: 与 MXC backend registry 对照（跨项目证据）
- **epistemic**: Cognitive Model（`Cross-project validation pending`）

## KO 聚合规则矩阵
| KO | 规则 | 簇内 EK | 边类型 | 可命名 | 可回溯 |
|----|------|---------|--------|--------|--------|
| KO-D01 | R1 | D02/D05/D19 | mechanism | ✅ | ✅ |
| KO-D02 | R3 | D04/D05/D06 | constraint | ✅ | ✅ |
| KO-D03 | R2 | D01/D02/D09/D10 | causal | ✅ | ✅ |
| KO-D04 | R4 | D04/D15 | subsystem/contrast | ✅ | ✅ |
| KO-D05 | R4 | D09/D12/D17 | mechanism/subsystem | ✅ | ✅ |
| KO-D06 | R1 | D07/D13/D14 | mechanism | ✅ | ✅ |

## 层级分布
- L3: 3（KO-D02/D03/D04）；L4: 3（KO-D01/D05/D06 全 `Cross-project validation pending`）；L5: 0（不硬凑）
- 聚合规则覆盖率 100%；无"同子系统=聚合理由"假聚合
