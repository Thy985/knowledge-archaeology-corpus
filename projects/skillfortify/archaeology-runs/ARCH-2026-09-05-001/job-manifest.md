# Daily Archaeology Job Manifest — 2026-09-05

> run_id: ARCH-2026-09-05-001 ｜ 日期：2026-09-05 ｜ 每日 1 个深度项目

## 1. KnowlegeMap 当前状态（2026-09-05 读取）
- **candidates**：27 张（S 6 / A 11 / B 5 / 首轮 4 + validated 2）
- **今日新增**（雷达 #4）：`[cand]skill-supply-chain-security-2026`（S）+ `[cand]agentic-graphrag-2026`（A）
- **已考古**（corpus COMPLETED）：deepseek-harness（09-03）、rampart（09-04）
- **Scan Logs**：05_scan_logs/ 共 8 次（bootstrap + mcp-evals + expansion + 雷达 ×4 + 09-05）
- **Connections**：04_connections 共 27 行，今日 +2

## 2. 候选排序（不按 star；加权：Knowledge Value / Agent-AI 相关性 / Novelty / 增长 / 近期发布 / Corpus 连接 / 已考古 / Refresh / Benchmark）
| # | 候选 | 等级 | 状态 | 核心理由 | 已考古 | 决策 |
|---|---|---|---|---|---|---|
| 1 | **SkillFortify**（skill-supply-chain 卡） | S | NEW | 今日雷达主线（Agent 供应链安全）；首个形式化验证框架（Dolev-Yao 模型 + 五定理 soundness）；与 RAMPART 形成安全攻防专题连续考古；FACT 证据；仓库可考古 | 否 | **✅ 选中** |
| 2 | Agentic GraphRAG（MemGraphRAG） | A | NEW | 补 G2 RAG 缺口；但为多工具主题卡，单项目考古价值低 | 否 | 次选 |
| 3 | Agent Harness Control Plane | S | QUEUED | 高价值概念卡；无明确单一仓库，且与 deepseek-harness 域重叠 | 否 | 待素材 |
| 4 | Agent Memory（Mem0 v3） | S | QUEUED | 高价值；主题卡需先确认单一仓库 | 否 | 待素材 |
| 5 | OWASP Agentic | S | QUEUED | 标准/文档类（www-project 仓库），非可考古代码 | 否 | 待素材 |
| 6 | A2A 协议 | S | QUEUED | 协议规范考古，Agent-AI 相关性中 | 否 | 待素材 |
| 7 | RAMPART | S | **COMPLETED** | ARCH-2026-09-04-001 | ✅ | 不选（refresh 不必要） |
| 8 | deepseek-harness | S | **COMPLETED** | ARCH-2026-09-03-001 | ✅ | 不选（refresh 不必要） |

## 3. 选中项目
```
job_id: ARCH-2026-09-05-001
project: skillfortify
repository: https://github.com/qualixar/skillfortify.git
mode: initial
priority: S
reason:
  - 今日雷达 #4 聚焦主线"Agent 供应链安全"（G1 相邻新前线）：SkillFortify 是首个形式化验证
    框架（Dolev-Yao 适配 DY-Skill 模型，五条数学定理保证 soundness），方法论上是
    "静态分析保证 vs 启发式扫描"的跃迁
  - 与昨日 RAMPART（ARCH-2026-09-04-001，运行时安全测试）+ deepseek-harness（dsh-pentest）
    形成"AI 安全攻防"专题的连续考古：运行时测试 → 供应链形式化验证
  - Agent-AI Engineering 相关性最高：Agent skills 供应链是 2026 最直接的 Agent 工程安全问题
    （OWASP ASI04 / AST01-10 直接对应）
  - Benchmark 学习价值高：形式化验证的 soundness 定理、能力静态分析、声明 vs 实际验证纪律，
    为 knowledge-archaeology-skill 的 L1-L5 升维提供全新素材
  - 仓库可考古且可达（qualixar/skillfortify HEAD dbb5942，+ skillfortifybench HEAD eb9d5a9）
  - 连接 EP-002（Permission Is Security Boundary）——skill 能力验证正是权限边界的声明侧保证
source_entry: 03_expansion_queue/candidates/[cand]skill-supply-chain-security-2026.md
```

## 4. 约束（本 run 遵守）
- KnowlegeMap 只读（不改候选卡状态）
- 每天 1 个项目；不覆盖 corpus 历史
- 飞书落盘默认不做（只 push corpus）
- 全部事实可追溯到仓库实际内容
