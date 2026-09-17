# Job Manifest — ARCH-2026-09-18-001

## 候选排序（不按 star 排名）
| 排名 | 项目 | 仓库 | Knowledge Value | Agent-AI 相关性 | Novelty | 增长/活跃 | 与 corpus 连接 | 状态 |
|---|---|---|---|---|---|---|---|---|
| 1 | **Letta（letta-code）** | letta-ai/letta-code | A（S 级卡核心对象；memory-first agent harness） | 极高（stateful agents / 自编辑记忆 / memory 内置化） | 高（自编辑记忆范式 vs 抽取式） | 高（09-17 推送，主仓 24.7K★ + code 3.3K★） | 强（memgraphrag 图记忆 / dsh-memory-evolve 记忆演化 → 记忆品类三线） | **NEW** |
| 2 | Mem0 | mem0ai/mem0 | A（抽取式记忆代表） | 高 | 中（候选卡已深度跟踪 09-13 架构细节） | 高（09-17 推送） | 与 memgraphrag 重叠较高 | QUEUED |
| 3 | agents-memory（Lolaplex） | Lolaplex/agents-memory | B（"记忆=Markdown 仓库"路线代表，与 KnowlegeMap 同构） | 中高 | 高（09-13 新） | 低（5★，378KB 微型） | 与 GrowthOS/KnowlegeMap 同构 | QUEUED |
| 4 | MemEval | ProsusAI/MemEval | B（评测套件，benchmark 学习价值） | 中高 | 中 | 低（52★，04-02 后停） | 与 agentevals/Validation 编译器连接 | QUEUED |
| 5 | agentmemory | hansonkim/agentmemory | B | 中高 | 中 | 低（0★，05-07 停更） | 编码 agent 记忆 | QUEUED |

**排除**：OpenClaw（09-17 已考古 COMPLETED，PR #17 open）；omnigent/opencode/rampart/clear/memgraphrag/dsh-memory-evolve/evolver/skillfortify/deepseek-harness/dogwood/agentevals（corpus 已有）；今日扫描无新 S/A/B 级对象需建卡（增量全部落入已有卡）。

## 选中项目
```yaml
job_id: ARCH-2026-09-18-001
project: letta-code
display_name: "Letta (letta-code) — memory-first stateful agent harness"
repository: https://github.com/letta-ai/letta-code.git
mode: initial
priority: A
reason:
  - "今日雷达（09-18）4 条 Agent Memory 密集增量（Apple SSM 选择性+共享 RBAC / MS AF Cosmos 原生记忆 / MemForest 时序 / OKF·Grok Build『记忆=Markdown 仓库』）全部命中 S 级 agent-memory 卡，记忆 harness 内置化路线成型"
  - "Letta 是 memory-first agent harness 的原型代表（MemGPT 系，自编辑记忆 + Context Constitution + Mods），与 corpus 已考古的 memgraphrag（图记忆）、dsh-memory-evolve（记忆演化）形成记忆品类第三条对照线"
  - "代码源已迁移至 letta-code（TypeScript，Apache-2.0，09-17 活跃），主仓 24.7K★ 但不可考古（main 已归档为文档仓，archive 分支含旧 V1）；letta-code 3.3K★ 是真实实现"
  - "Agent-AI Engineering 相关性极高：stateful memory / memory confinement / frontmatter / mods / skills / sandbox 权限，直接服务用户 GrowthOS/TeamMind 记忆设计"
  - "Benchmark 学习价值：MemEval/LoCoMo/LongMemEval 批判（09-13/09-18 增量）可对照自编辑记忆 vs 抽取式记忆的评测口径"
source_entry:
  - "KnowlegeMap 03_expansion_queue/candidates/[cand]agent-memory-2026.md（S 级，09-18 增量）"
  - "KnowlegeMap 05_scan_logs/2026-09/2026-09-18_radar-watch_scan.md §2 信号 4-6"
corpus_slug: letta-code
snapshot:
  commit: 3df2ebd6e5826b482f0124e2b02f0e4e0f1af567
  commit_message: "docs(skills): use applicability metadata in letta-guide (#4515)"
  branch: main
  version: 0.32.12
  timestamp: 2026-09-17T10:40:54-07:00
  size: 93MB
  files: 2308 (1871 .ts / 146 .md)
  language: TypeScript (Bun)
  license: Apache-2.0
```

## 聚焦问题
1. Letta 如何实现"记忆像人一样"的持久化——memory blocks / memory confinement / frontmatter / 自编辑？
2. 记忆写入-检索-演化回路（recall / archival / core memory）的真实控制流与状态流？
3. Mods / skills / sandbox / permissions 如何构成 harness 治理面？记忆与权限边界的关系？
4. "记忆=文件/Markdown 仓库"路线（OKF/Grok/Anthropic /mnt/memory）与 Letta 的对照，是否支撑 09-18 雷达假说？

## 硬约束
- 只读目标项目；不改 KnowlegeMap / 生产 Skill；不覆盖 corpus 历史 run
- 每天最多深度完成 1 个项目（本日唯一：letta-code）
- 飞书不落盘（未授权）；不执行 Skill Evolution（除非暴露系统性缺陷）
