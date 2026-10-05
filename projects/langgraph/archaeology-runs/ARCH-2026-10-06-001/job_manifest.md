# Job Manifest — ARCH-2026-10-06-001

## 1. 决策摘要
- **job_id**: ARCH-2026-10-06-001
- **project**: langgraph
- **repository**: https://github.com/langchain-ai/langgraph.git
- **mode**: initial
- **priority**: A
- **source_entry**: `[cand]multiagent-framework-2026.md`

## 2. 候选排序（SOP 九维度，非 star 排名）
本轮从 KnowlegeMap 38 张候选卡中按九维度（Knowledge Value / Agent-AI Engineering 相关性 / Novelty / 增长信号 / 近期发布 / 与已有 Corpus 连接 / 是否已考古 / 是否需要 refresh / Benchmark 学习价值）排序，剔除已考古或域覆盖卡：

| 排名 | 候选 | 卡级 | 决策理由（摘要） | 状态 |
|---|---|---|---|---|
| 1 | **langgraph**（multiagent 卡代表） | B→**A** | 2026 企业多 agent 默认框架（v1.0 GA，34% 千人员工企业文档出现率）；图编排+supervisor+checkpointing 为 Agent-AI 核心可沉淀模式；corpus 无 graph-based 编排框架（Novelty 高）；雷达 next-step 第一优先（TeamMind 隔离 vs 共享取舍）直接需要其状态设计对照；42.7k stars（非排序依据）、MIT、10-05 仍活跃 | **选中** |
| 2 | agent-formal-verification 卡 | A | 代表仓库 aprover 已于 09-28 考古（无新 commit）；10-06 增量（VeriHarness/AgentVerify/Specula/Lean4Agent）均无可考古 GitHub 仓库（4 探测 404） | COMPLETED/无仓库 |
| 3 | agentic-inference-infra 卡 | A | 代表仓库 NVIDIA Dynamo 已于 09-20 考古（ai-dynamo/dynamo）；10-06 增量（KVCacheStore/vllm-metal/TensorRT Edge-LLM）为产品/博客非仓库 | COMPLETED/无仓库 |
| 4 | flutter-local-llm 卡 | A | 用户 FixMath 相关（Dart 端侧 LLM），代表仓库 llamafu 为 FFI 包装（llama.cpp），Agent-AI 工程内核薄；留待月度轮 | QUEUED |
| 5 | local-edge-llm 卡 | A | 月度轮；代表多产品（GLM-Edge/Liquid/RTX Spark）无单仓库 | QUEUED（月度） |
| 6 | pi-hunter / agentic-attack | A/S | 论文族/事件线，无仓库可考古（pyrit 已覆盖 offensive 域） | QUEUED/无仓库 |
| 7 | computer-browser-use / agent-memory / ai-offensive / agent-firewall / otel-genai / agentic-benchmarks / open-source-agentic-ci 等 | A/S/B | 域已被 corpus 考古覆盖（browser-use/browser-harness/webwright；mem0/letta；pyrit；guardian/aigis；langfuse；agentevals；pullfrog） | COMPLETED（域覆盖） |

## 3. 选中项目理由（LangGraph）
1. **Knowledge Value（高）**：2026 年企业多 Agent 编排的事实默认（LangGraph v1.0 GA；34% 的 1000+ 人企业生产架构文档中出现——multiagent 卡实测数据）；有状态图 + supervisor + checkpointing + interrupt(HITL) 是可长期复用的 Agent 架构模式。
2. **Agent-AI Engineering 相关性（极高）**：直接回答雷达 next-step 第一优先"TeamMind 多 agent 状态设计（隔离 vs 共享）取舍"——LangGraph 的共享状态图 vs OpenClaw 的 workspace 隔离构成对照两极；checkpoint 机制是"状态持久化"范式参考。
3. **Novelty（高）**：corpus 现有 omnigent（harness）、openclaw（CLI agent）、agent-governance-toolkit（治理）、a2a（协议）、letta-code/mem0（记忆）——均非"图编排执行引擎"层；LangGraph 是首个 graph-orchestration 对象。
4. **增长信号 / 近期发布**：v1.0 GA（2026 Q1）；仓库最后 push 2026-10-05（活跃）；v0.x→v1.0 语义稳定化。
5. **与已有 Corpus 连接**：checkpointing ↔ letta-code/mem0（记忆持久化）；tool node ↔ opencode/browser-harness（工具调用协议）；supervisor ↔ omnigent/agent-governance-toolkit（治理层对照）。
6. **是否已考古**：否（corpus 无 langgraph 目录）。
7. **是否需要 refresh**：否（initial）。
8. **Benchmark 学习价值**：Pregel 式图执行引擎、checkpoint 状态快照、interrupt/resume 循环、supervisor 图——大量可提炼为 EK 的确定性机制。

## 4. 执行约束
- 只读克隆（depth 1）至 `archaeology-jobs/ARCH-2026-10-06-001/repo/`
- 生产版 skill v3.2；每天最多 1 个深度项目；不覆盖 corpus 历史
- 无 Skill Improvement Proposal 预期（若暴露系统性缺陷则走 Evolution 流程）
