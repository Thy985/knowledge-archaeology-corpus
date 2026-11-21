# dapr-agents 知识考古索引

> 一句话定位：**分布式 Agent 运行时框架**——DurableAgent 把 LLM 推理与工具调用包进 Dapr durable workflow（每步持久化 + 崩溃恢复不重放），HITL 审批默认 fail-closed，hooks/lifecycle 双轨扩展，MCP 零配置自动发现（CNCF Dapr × NVIDIA，2026-10 launch）。

## 考古信息

- 仓库：https://github.com/dapr/dapr-agents.git（commit `8b4cc8d`，v1.0.0，2026-10-22，Apache-2.0）
- 考古日期：2026-10-25 · run_id：`ARCH-2026-10-25-001` · skill：knowledge-archaeology v3.2
- 三仓库边界：本仓（被考古对象）/ KnowlegeMap（发现层）/ knowledge-archaeology-corpus（结果库）

## 产物清单

| 目录/文件 | 内容 |
|---|---|
| `metadata.yaml` | 考古元数据（commit/run/summary/质量指标/判定统计） |
| `00_overview.md` | 项目总览（六条认知主线） |
| `01-project-layer/project_layer.md` | 项目地图（架构/模块/生命周期/数据结构/配置/治理） |
| `02-engineering-knowledge/ek_graph.md` | EK Graph 20 条（六类边 links 100%） |
| `03-knowledge-layer/knowledge_layer.md` | Generalized KO 6 条（R1-R4 聚合规则） |
| `04-flow-atlas/flow_atlas.md` | 七类流（Control/State/Data/Evidence/Authority/Memory/Policy） |
| `05-candidates/candidates.md` | 未验证假说 11 条（Hypothesis，含跨项目 C-D05） |
| `06-validation/` | Validation 报告 + Independent Validation Report（13 判定）+ Reconciliation（R-1..R-5） |
| `archaeology-runs/ARCH-2026-10-25-001/` | 本轮原始产物（job_manifest/snapshot/run_metadata/IVR） |

## 关键发现（摘要）

1. **durable execution 三件套**（KO-D01 L4）：持久化点 + 幂等活动（call_llm 重放检测 task_execution_id）+ 确定性 ID（approval uuid5）——缺一，崩溃恢复即重放或重复
2. **HITL 默认 fail-closed**（KO-D02 L3）：超时 auto-deny / 解析失败 auto-deny / 无交互通道短路；仅显式 None 才无限挂起
3. **状态三层分离**（KO-D05 L4）：execution state（workflow）≠ session memory（manifest+chunk）≠ tool history（观测）——OpenAI 消息链约束被显式处理（skip_messages 双模式）
4. **扩展点双轨**（KO-D04）：hooks 决策型（五态）+ lifecycle Protocol 拦截型（OSS-clean）
5. **治理代码化**：AGENTS.md 提交门 + DCO + agent-metadata schema 版本化（v0.11.1→v1.0.0）+ schema-compat CI

## 关键争议

- C-D05（跨项目假说）：dapr "确定性三件套" 与 MXC/strands-box fail-closed 链条是同一治理模式两端（可靠 vs 遏制）——未验证前不升 Principle
- 反例 3 成立 + 1 部分：KO-D01 的"可靠不含可终结"（None 挂起）、KO-D02 显式放弃 fail-closed、KO-D06 zero-config 反例
- Auditor 3 修正：EK-D18 证据锚 / EK-D13 实现文件 / EK-D04 Mutate 语义——全部合入（R-1..R-3）

## Skill Improvement Proposal

- 无 SIP（本轮未暴露系统性 skill 缺陷；EK Graph/R1-R4/盲重建流程运行正常）
