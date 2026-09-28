# Job Manifest — ARCH-2026-09-29-001

## 候选排序（SOP 标准，非 star 排名）

| 排序 | 候选（候选卡） | Knowledge Value | Agent-AI 相关性 | Novelty | 增长/近期 | Corpus 连接 | 已考古 | 结论 |
|---|---|---|---|---|---|---|---|---|
| 1 | **microsoft/agent-framework**〔agent-harness-control-plane〕 | 高：生产级 agent 框架（Workflows/Agents/多代理编排/原生记忆） | 极高 | 中（1.0 GA 后） | push 2026-09-28（极活跃, 136MB 大仓） | 强：omnigent(meta-harness)/deepseek-harness/letta-code 三形态对照 | 未考古 | ✅ **选中** |
| 2 | Tencent BrowserSkill（computer-browser-use-2026） | 中高：浏览器 agent 桥（MIT） | 高 | 中 | 06-2026 开源 | 中：browser-use（已考古）对照 | 未考古 | 备选（需定位确切仓库） |
| 3 | Lolaplex/agents-memory（agent-memory-2026） | 中：本地 markdown 记忆验证"文件仓库"路线 | 中高 | 高（观察级） | push 2026-09-26 | 强：KnowlegeMap/GrowthOS 文件记忆路线同构 | 未考古 | 备选（★5 规模小, 深度有限） |
| 4 | agent-memory 论文族（Synapse/REMem/HeLa-Mem/SEEM/Jev-Mem） | 高：记忆研究新范式 | 高 | 高 | 09-29 大增量 | 中 | 非仓库 | ❌ 全为 arXiv 论文, 无考古仓库对象 |
| 5 | mcp-notification 系列（angelcervera/mcp-notification 等） | 低中：通知 server 单点实现 | 中 | 低 | push 2026-01（停滞） | 中：agent-attention 同域 | 未考古 | ❌ 单仓库过小, 停滞 |

## 选中项目

- **job_id**: ARCH-2026-09-29-001
- **project**: agent-framework（MS Agent Framework）
- **repository**: https://github.com/microsoft/agent-framework.git
- **mode**: initial
- **priority**: P1
- **reason**: ① 雷达 09-29"下一步"第一条焦点 = Agent Harness（MS AF 1.0 是生产代表，Agent Harness/Hosted Agents/CodeAct/多代理编排在 2026 Build GA）；② 昨天已核验：★13841、push 2026-09-28（当日活跃）、MIT、Python/C# 双栈、136MB 生产级大仓；③ Corpus 对照价值：omnigent（meta-harness，已考古）→ MS AF（生产框架）→ deepseek-harness（内部 harness）构成"harness 三形态"对照；④ Agent Memory 09-04 增量（Cosmos 原生记忆 CosmosMemoryContextProvider）挂 MS AF——"记忆→harness 内置化"趋势实例（09-18/09-21 雷达主线）；⑤ 多代理编排模式（sequential/parallel/round_robin/agent_team）是 Agent-AI Engineering 核心知识，与 Harness 六职责（observation/context/control/action/state/verification）互证。
- **source_entry**: KnowlegeMap `03_expansion_queue/candidates/[cand]agent-harness-control-plane.md`（MS Agent Framework 条目）+ `[cand]agent-memory-2026.md`（Cosmos 原生记忆增量）+ 雷达 09-29"下一步"
