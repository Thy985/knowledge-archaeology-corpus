# Job Manifest — ARCH-2026-10-08-001

| 字段 | 值 |
|---|---|
| job_id | ARCH-2026-10-08-001 |
| project | ThinkingBox |
| repository | https://github.com/microsoft/thinkingbox.git |
| mode | initial |
| priority | P0 |
| source_entry | KnowlegeMap `03_expansion_queue/candidates/[cand]agentic-benchmarks-2026.md`（2026-10-08 雷达增量：ThinkingBox 执行真相评测） |

## 1. 候选排序（九维，非 star 排名）
| 排序 | 候选 | 知识价值 | Agent-AI 相关性 | 新颖性 | 增长信号 | 近期发布 | Corpus 连接 | 已考古? | Benchmark 价值 | 结论 |
|---|---|---|---|---|---|---|---|---|---|---|
| **1** | **microsoft/thinkingbox** | A（评测范式转折） | 极高（MCP 工具+agent 运行+行为评测三合一） | 高（10-03 发布 5 天） | 95★/10 forks（发布即起） | ✅ pushed 2026-10-03 | evals/agentevals/deepseek-harness（评测线）+ mcp（工具线）+ owasp-agentic-skills | 否 | 极高（自带可执行评测框架） | **选中 P0 initial** |
| 2 | open-telemetry/opentelemetry-python-contrib | B（标准实现） | 中（观测标准） | 中（增量 OTel 2.0 Run Tree） | 1106★ | ✅ | langfuse（OTel 实现侧已考古） | 否 | 中 | 备选：范围庞大（多 instrumentor），增量价值被 langfuse 覆盖 |
| 3 | a2a refresh | B（协议线） | 高 | 低（增量 AAIF governance） | — | 10-04 增量 | a2a 已考古（09-24） | **是** | 低 | refresh 条件未满足（无大版本变化证据） |
| 4 | otel-genai 代表仓库（阿里/Google） | C | 中 | 低 | — | 404 | — | — | 低 | 探测 404（无独立代码仓库） |

## 2. 选中理由（九维详细）
1. **Knowledge Value A**：ThinkingBox 按"数据库最终状态+副作用"评分而非工具调用/最终回答——121,680 trials × 12 模型：79,853 次未过可执行检查，**67.24% 失败仍"干净结束、无任何报错"**，**79.9% 失败源于工具处理**——"agent 撒谎完成任务"被系统量化。这是 2026 评测最关键范式转折（pass@1 幻觉终结），直接指导 silver-shield Harness 引入"执行状态断言层"（雷达验证优先级第一）
2. **Agent-AI Engineering 相关性极高**：仓库自述 "framework for defining tool as MCP servers, running LLM agents against them, and evaluating agent behavior"——工具定义（MCP）× agent 运行 × 行为评估三合一，正是 harness/评测/工具契约三条线的交点
3. **Novelty**：2026-10-03 发布（Microsoft Research + Hugging Face 背书），5 天前
4. **增长信号**：发布即 95★/10 forks（相对早期信号）
5. **近期发布**：pushed_at 2026-10-03（与雷达增量同日）
6. **与已有 Corpus 连接**：直接连 evals/agentevals/deepseek-harness（评测与验证编译器线）——mcp（工具定义线，09-25 考古）——owasp-agentic-skills（09-26，评测+安全交叉）——langgraph（10-06，MCP 生态）
7. **未考古**：corpus 28 项目无 thinkingbox
8. **无需 refresh**
9. **Benchmark 学习价值极高**：自带可执行评测框架（MCP 工具 + 多轮 trials × 多模型），其"状态真相评分"是可独立复现的 Benchmark case

## 3. 排除说明
- otel-genai 各候选仓库（google/agent-observability、alibaba/opentelemetry-genai-utils、AgentsLastExam/benchmark）API 探测 404——增量多为文档/平台能力声明，无独立代码仓库；OTel 实现侧已由 langfuse（10-01）覆盖
- a2a：已考古（09-24），10-04 AAIF governance 增量不足以触发 refresh（无版本主线大变化证据）
- 其他 35 卡：无 10-08 增量或已考古/无代表仓库

## 4. 预期考古范围
- MCP server 工具定义机制 + agent 运行 loop + 评测评分（状态/副作用真相）三模块
- 67.24% 静默失败与 79.9% 工具处理失败的可复现性（若仓库含数据/脚本）
- 与 mcp/agentevals/deepseek-harness corpus 对象的跨项目连接
