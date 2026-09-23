# Job Manifest — ARCH-2026-09-24-001

| 字段 | 值 |
|---|---|
| job_id | ARCH-2026-09-24-001 |
| project | A2A（Agent2Agent 协议） |
| repository | https://github.com/a2aproject/A2A.git |
| mode | initial |
| priority | S（KnowlegeMap 候选卡明确 S 级，未深入、未验证） |
| date | 2026-09-24 |

## 候选排序（多因子，非 star 排名）

| rank | 项目 | ★ | pushed | 因子评估 | 状态 |
|---|---|---|---|---|---|
| 1 | **A2A** | 25,907 | 2026-09-22 | Knowledge Value 高（多 Agent 互操作协议层，TeamMind 协议层直接相关）；Novelty 高（协议/互操作考古空白）；增长信号强（v1.1、13 框架集成、Java SDK fail-closed 授权、Azure Foundry GA）；Benchmark 价值高（fail-closed 授权、任务生命周期、信任模型） | NEW |
| 2 | MCP servers | 90,565 | 2026-09-22 | 生态声望高但主体是参考实现集合，协议规范考古价值分散；Skills over MCP Final 是协议层议题非仓库实现 | NEW（次选） |
| 3 | hermes-agent | 248,338 | 2026-09-23 | 极活跃，但历史已有考古（open PR #19 未合并 main），同族（Agent Framework）已有 openclaw/letta-code/omnigent 考古，边际递减 | COMPLETED（PR 待合并） |
| 4 | browser-use | 116,066 | 2026-09-18 | 历史已有考古（open PR #22），同族边际递减 | COMPLETED（PR 待合并） |
| 5 | StarHarness | 3 | 2026-08-26 | 雷达今日最高价值指向（harness 自演化）但仓库规模过小，不足支撑深度考古 | NEW（暂缓） |
| 6 | Webwright | 6,017 | 2026-08-03 | B 级弱活跃，一手来源已核验但无考古增量 | QUEUED |
| 7 | RAMPART | 415 | 2026-09-22 | corpus 已有 rampart 考古 | COMPLETED |

## 选中：A2A

**reason**：
1. KnowlegeMap 候选卡 A2A = S 级、[cand]、未深入、未验证——雷达 09-23 增量明确"v1.1 + 13 框架内置集成 + Java SDK 默认 fail-closed 授权"，处于实现成熟期
2. 与用户 Corpus 连接：TeamMind（协议层）、MCP 互补、OpenClaw/omergent 互操作对照——可形成跨项目模式验证
3. 品类轮换：沙箱（E2B 09-23）→ 协议/互操作（A2A 今日），避免同族边际递减
4. Benchmark 学习价值：fail-closed 授权默认值、任务生命周期状态机、AgentCard 发现机制——Agent 系统安全与互操作的高价值证据

**source_entry**：KnowlegeMap `06_expansion_index/README.md` —— 2026-09-01 首轮扫描「A2A 协议 v1.0 + AAIF」[cand] S；雷达增量 09-02/09-16/09-23
