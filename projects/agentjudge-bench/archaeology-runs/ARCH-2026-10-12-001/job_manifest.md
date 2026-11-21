# Job Manifest — ARCH-2026-10-12-001

> 每日 Knowledge Archaeology Runtime（SOP）｜ 2026-10-12（Monday）｜ run-id: `ARCH-2026-10-12-001`
> 数据源：KnowlegeMap（origin/main `faf84fd`，10-12 雷达第 40 次扫描）

## 一、KnowlegeMap 当前状态摘要

- **领域状态**：Evaluation / Observability 周**正式启动**（active，10-12 起）；周一分工=建领域能力树/基线，**无新候选卡**（周一不分推增量）
- **能力树产出**：`08_agent_centric/domains/agent-evaluation.md`（A 评估设计 / B 执行观测 / C 评测执行 / D 判定打分 / E 学习演进 五层）
- **S 级缺口**：**评测环境隔离设计**（网络/凭据/外部目标三隔离，10-11 重大事件驱动）
- **本周验证计划（周五 10-16）**：① AgentJudgeBench 精读对比（10-13 前）② 评测隔离设计落地检查（DeepSeek Harness·silver-shield）③ promptfoo Agent Evals 扩展实测
- **候选卡**：39 张（[cand]）+ 2 validated；相邻素材记日志未建卡：AgentSeer（arXiv 2509.04802）、HAL（Princeton Sayash Kapoor）

## 二、候选排序（九维，非 star 排名）

| 排序 | 候选 | 状态 | 仓库可得性 | 九维评估（价值/相关性/新颖/增长/发布/连接/已考古/refresh/benchmark） |
|---|---|---|---|---|
| **P0** | **AgentJudgeBench** | QUEUED（A 级卡，10-13 前精读对象） | ✅ ServiceNow/SyGra 分支 `scratch/agent_judge_bench`（`97cca4f`） | 知识价值 A（LLM-as-Judge 可靠性量化）；Agent-AI 相关性极高（评测信任核心）；新颖（judge 本身评测）；近期（08-27 arXiv/2608.26623）；连接引擎 evaluation 层/silver-shield/promptfoo；未考古；initial；Benchmark 学习价值高（程序化 judge vs LLM judge 对照方法论） |
| P1 | promptfoo | validated 锚点（本机 2/2 PASS） | ✅ promptfoo/promptfoo | 价值 B；相关性高；未考古；但已 validated 且本周是"扩展实测"而非深考古 |
| P2 | silver-shield | 用户自有连接项目（防诈骗/风险检测），非候选卡 | ❌ 无公开仓库（私有） | 价值 A 但**不可浅克隆**（SOP 关键路径要求目标项目浅克隆）；留待用户授权 |
| P3 | AgentSeer / HAL | 素材级（10-12 记日志不建卡） | HAL 无公开仓库（探测 404）；AgentSeer 未核实 | 本周素材，非候选 |
| 排除 | agentic-attack / agentjudge 其他同名 | 无代码仓库 | — | 10-11 轮已排除；AgentJudgeBench 代码实为 ServiceNow/SyGra 分支，本轮实测可得 |

## 三、选中项目

```
job_id: ARCH-2026-10-12-001
project: agentjudge-bench
repository: https://github.com/ServiceNow/SyGra.git
branch: scratch/agent_judge_bench
commit: 97cca4fa07e98ee61bb3a1a2f8639778e299d50a
mode: initial
priority: P0
reason: >-
  Evaluation 周（10-12 起）S 级主题"评测可信"的核心候选；A 级卡 + 10-13 前精读对象；
  AgentJudgeBench 是 LLM-as-Judge 可靠性量化的权威基准（3,808 记录 × 3 难度 × 5 生成器
  × 6 judge × 2 条件 = 321,648 次评测），与引擎 evaluation 层/silver-shield/promptfoo
  同线连接；程序化 judge（Eqs 1-3）+ alignment（Eq 4）的设计是"证据→判断"可靠性理论层
  的工程实例，Benchmark 学习价值高。代码实测可得（SyGra scratch/agent_judge_bench 分支）。
source_entry: "03_expansion_queue/candidates/[cand]agentjudge-bench-2026.md（10-10 雷达 A 级）；
  08_agent_centric/domains/agent-evaluation.md（10-12 能力树：D1 LLM-as-Judge 可靠性缺口 A 级）；
  论文 arXiv 2608.26623 原文代码指针 ServiceNow/SyGra/tree/scratch/agent_judge_bench"
```

## 四、执行计划映射（六阶段）

1. ✅ 阶段① 权限核验 PASS（三仓库 dry-run up-to-date；PAT 有效）
2. ✅ 阶段② Job Selection（本 Manifest）
3. ⏳ 阶段③ Snapshot（已浅克隆 45MB/436 文件；HEAD `97cca4f` "AgentJudgeBench related files"）
4. ⏳ 阶段④ 正式 Archaeology（v3.2 多 Agent；核心对象 tasks/agentic_bfcl_judge_eval/ + sygra 骨架）
5. ⏳ 阶段⑤ 独立盲重建（攻击：程序化 judge 公式→Pattern、难度→L4、GT anchoring→通用 Principle、Flow edge）
6. ⏳ 阶段⑥ Reconciliation + Corpus Commit + PR（corpus 新项目 `projects/agentjudge-bench/`）
