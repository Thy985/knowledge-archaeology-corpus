# Job Manifest — 2026-10-13（Evaluation 周 Day2）

- **run_id**: `ARCH-2026-10-13-001`
- **选择时间**: 2026-10-13（UTC+8）02:20
- **KnowlegeMap 状态**: origin/main `7c232a7`（radar watch 2026-10-13 Day2，第 41 次扫描，候选卡 39→40）

## 候选排序（九维评估，非 star 排名）

| 排名 | 项目 | 状态 | 代码可得性 | 理由 |
|---|---|---|---|---|
| **P0** | **Transect** | **NEW**（今日唯一新候选卡） | ✅ `github.com/AI-Safety-Institute/transect` main=`75a7948` | Evaluation 周 Day2 增量 A 级卡（arXiv 2610.08364，10-06 v1/10-08 v2）；评测者侧**可审计性**（标签→源头回合可追溯 + 数据表导出），与昨日 AgentJudgeBench（judge 侧可靠性）互补成"评测可信度双环"；能力树 B 层直接对应；与引擎 evaluation 层（criterion+证据保留）有直接连接；UK AISI 官方（Inspect 生态）背书；增长信号强（v1→v2 两天 + 活跃 PR≥14） |
| P1 | promptfoo | QUEUED（validated 锚点） | ✅ 可得 | 已 validated，按 SOP 不做深度项目；保持锚点 |
| P2 | silver-shield | QUEUED | ❌ 无公开仓库 | 无专属仓库/候选卡，无法深度考古 |
| P3 | AgentSeer / HAL | QUEUED | ❌ 404 | 10-10 已记日志但无可考古仓库 |
| — | OpenLIT/Iris/LoongSuite/Laminar | 去重 | — | 已在 [cand]otel-genai-observability.md 或既有格局内，不建卡不考古 |

## 选中项目

- **project**: Transect（长时程 Agent 评测可观测性/可审计性包）
- **repository**: https://github.com/AI-Safety-Institute/transect.git
- **mode**: initial
- **priority**: P0
- **reason**:
  1. **Knowledge Value（A）**: 解决"评测可观测性收缩"（observability envelope）——长时程 agent 评测 transcript 可达 1300 万 token，LLM 转录分析给评测者显著分析自由度，威胁可复现/可审计性；提供"评估族配置 + judge 分离 + 回合时间线 + 标签溯源 + 数据表导出"完整方案
  2. **Agent-AI Engineering 相关性（高）**: 直接服务引擎 evaluation 层（criterion 打分 + 证据保留 → 回合时间线 + 标签可追溯升级路径）；与昨日 AgentJudgeBench 结论（judge 对齐随难度下降）配套——长时程 judge 输出必须有可追溯结构
  3. **Novelty（高）**: 10-06 v1 / 10-08 v2，评测可信度热点期（ThinkingBox 10-08、AgentJudgeBench 10-10、Transect 10-13）最新一环
  4. **增长信号（强）**: v1→v2 仅 2 天（活跃修正）；GitHub 已有 ≥14 个 PR；UK AISI 官方维护（Inspect 生态轨道）
  5. **Benchmark 学习价值（高）**: 与 AgentJudgeBench 并读可形成"评测可信度双环"（judge 可靠性 + 审计可追溯）验证小结，回灌能力树 B/D 层
- **source_entry**: `KnowlegeMap 03_expansion_queue/candidates/[cand]transect-agent-eval-observability-2026.md`（2026-10-13 雷达第 41 次扫描增量；论文一手证据 arXiv 2610.08364 正文明确代码指针 `github.com/AI-Safety-Institute/transect`，实测可得）

## 状态判定依据

- **NEW**: Transect——今日扫描新建卡，corpus 无 `projects/transect`（duplicate check）
- **QUEUED**: promptfoo（validated 锚点）/ silver-shield / AgentSeer / HAL
- **COMPLETED（昨日）**: AgentJudgeBench（ARCH-2026-10-12-001，PR #41 open）
- **REFRESH_REQUIRED**: 无（各项目最近考古在本周内）
- **IN_PROGRESS**: 无
