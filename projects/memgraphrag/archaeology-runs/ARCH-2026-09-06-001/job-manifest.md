# Daily Archaeology Job Manifest — 2026-09-06

- **run_id**: ARCH-2026-09-06-001
- **date**: 2026-09-06（SOP 定时任务轮次，cron 11659931249922）
- **job_id**: ARCH-2026-09-06-001
- **project**: memgraphrag
- **repository**: https://github.com/xmudeeplit/memgraphrag.git
- **mode**: initial
- **priority**: A
- **source_entry**: `03_expansion_queue/candidates/[cand]agentic-graphrag-2026.md`

---

## 候选排序（2026-09-06，按 9 维综合，不按 star）

| 排名 | 候选 | 等级 | 状态 | Knowledge Value | Agent-AI 相关性 | 是否可深度考古 | 与 Corpus 连接 | 结论 |
|------|------|------|------|----------------|----------------|----------------|----------------|------|
| 1 | **MemGraphRAG**（xmudeeplit/memgraphrag） | A | QUEUED（昨日次选） | 高：KDD'26 三层记忆+多 Agent 图构建，补 RAG/记忆领域缺口 | 高：Agent 记忆×检索融合 | ✅ 单一 MIT 代码仓库（HEAD cd6fabd 已核验） | 新专题（与现有 3 个 AI 安全项目互补） | **选中** |
| 2 | Agent Harness Control Plane（Omnigent+MS Agent Framework） | S | QUEUED | 高但为领域调查 | 高 | ❌ 多项目体系，非单仓库 | 弱 | 留待后续 |
| 3 | Agent Memory 2026 实现层 | S | QUEUED | 高但为全景综述 | 高 | ❌ 综述类 | 弱 | 留待后续 |
| 4 | OWASP Agentic Security | S | QUEUED | 高但为标准体系 | 高 | ❌ 标准文档类 | 中 | 留待后续 |
| 5 | A2A Protocol | S | QUEUED | 中：协议标准 | 高 | ❌ 协议规范类 | 弱 | 留待后续 |
| 6 | Agentic UX 2026（今日雷达新增） | B | NEW | 中：设计模式体系 | 中 | ❌ 模式清单类 | 弱（TeamMind/campus_order） | 留待后续 |
| 7 | skillfortify | S | COMPLETED（09-05） | — | — | — | corpus PR #3 已合并 | 已考古 |
| 8 | rampart | S | COMPLETED（09-04） | — | — | — | corpus PR #2 已合并 | 已考古 |
| 9 | deepseek-harness | S | COMPLETED（09-03） | — | — | — | corpus PR #1 已合并 | 已考古 |

## 选择理由（MemGraphRAG）

1. **Knowledge Value（高）**：KDD'26 接收，三层记忆（unstructured passages / extracted facts / abstract schemas）+ 分层索引图 + 多 Agent 组协同构建，是 2026 RAG 从"静态检索"走向"agent 主动遍历"的收敛点；填补 KnowlegeMap G2 RAG 系统化缺口（KB 仅 1 篇）。
2. **Agent/AI Engineering 相关性（高）**：多 Agent 协同构建图 + 记忆感知分层检索，直接命中 Agent Memory（S4）与 Context Engineering 主题；与 corpus 已有 3 个项目（deepseek-harness / rampart / skillfortify）形成领域互补（评估→安全→记忆/检索）。
3. **Novelty（高）**：三层记忆架构 + 多 Agent 构建管线的工程实现是首次考古对象（此前均为安全/评估领域）。
4. **增长/近期信号**：KDD'26 接收 + arXiv 2606.00610（2026-05-27）+ 2026-08 医学实证论文引用。
5. **与用户项目连接**：GrowthOS（三层记忆同构）、TeamMind（多 Agent 编排）、silver-shield（检索升级）——考古产出可直接映射个人项目。
6. **Benchmark 学习价值**：多 Agent 图构建管线的"agent 编排 + 证据保留（provenance）"实现是 benchmark 候选。

## Job Manifest

```yaml
job_id: ARCH-2026-09-06-001
project: memgraphrag
repository: https://github.com/xmudeeplit/memgraphrag.git
mode: initial
priority: A
reason:
  - "昨日次选；Agentic GraphRAG 收敛点（三层记忆+多Agent图构建）"
  - "补 corpus RAG/记忆领域缺口（现有3项目均为AI安全/评估）"
  - "KDD'26 接收 + arXiv 2606.00610，近期活跃"
  - "与 GrowthOS/TeamMind/silver-shield 直接连接"
  - "多Agent编排+证据保留实现为 benchmark 候选"
source_entry: 03_expansion_queue/candidates/[cand]agentic-graphrag-2026.md
```

## 约束确认

- 只选 1 个项目 ✅
- 不修改 KnowlegeMap / 生产 Skill / 目标项目 ✅
- 每日最多 1 个深度项目 ✅
- 只 push corpus，飞书落盘需授权（默认不做）✅
