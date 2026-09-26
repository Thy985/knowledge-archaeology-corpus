# Job Manifest — ARCH-2026-09-23-001

## Candidate Ranking Summary（多因子排序，非 star 排名）

| 排名 | 项目 | ★ | pushed_at | 状态 | Knowledge Value | Agent/AI 相关性 | Novelty | 增长信号 | 与 Corpus 连接 | 已考古 | 判定 |
|---|---|---|---|---|---|---|---|---|---|---|---|
| 1 | **e2b-dev/E2B** | 13,920 | 2026-09-22 | NEW | A（代码沙箱/安全执行，G1 治理主线） | 极高（TeamMind/EP-002/dsh-pentest） | 高（corpus 无沙箱品类） | 强（昨日仍 push） | omnigent/deepseek-harness/browser-use(sandbox 对照) | 否 | **选中** |
| 2 | browser-use/browser-harness | 18,023 | 2026-09-12 | NEW | B（自愈 harness，与 browser-use 同族） | 高（harness 自愈） | 中（同族第二考古边际递减） | 中 | browser-use（昨日考古） | 否 | 候选 |
| 3 | getzep/graphiti | 31,078 | 2026-09-21 | NEW | B（时序知识图谱） | 中 | 低（memgraphrag/mem0 同域连续考古） | 中 | memgraphrag/mem0 | 否 | 候选 |
| 4 | microsoft/Webwright | 6,015 | 2026-08-03 | QUEUED | B（SOTA browser agent） | 中 | 低（browser-use 昨日考古） | 弱（1.5 月未推） | browser-use | 否 | 候选 |

**排除逻辑**：browser-harness 与昨日 browser-use 同属浏览器团队同族卡（同项目族只考古一次原则，浏览器主线已覆盖）；graphiti 与 memgraphrag/mem0（09-21 考古）记忆/图谱同域，连续品类边际递减；Webwright 活跃度弱（08-03 后无推送）且与 browser-use 同域。

**选择逻辑**：E2B 是 **A 级卡「E2B 沙箱 + APP」** 的对象——代码沙箱/安全执行环境是 Agent 安全治理（G1 战略主线）与用户项目（TeamMind 沙箱执行/EP-002 供应链安全/dsh-pentest 安全测试）的直接连接点；corpus 无沙箱主题项目（品类轮换：09-21 记忆 → 09-22 computer-use → 09-23 沙箱隔离）；活跃度最高（昨日仍有 push）。

## Job Manifest

```yaml
job_id: ARCH-2026-09-23-001
project: e2b
repository: https://github.com/e2b-dev/E2B.git
mode: initial
priority: A
reason:
  - "A 级卡「E2B 沙箱 + APP」对象：开源企业级 agent 安全执行环境（sandbox + SDK + 托管基础设施）"
  - "Knowledge Value A：代码沙箱/隔离执行是 Agent 安全治理（G1）核心主题，corpus 空白品类"
  - "直接连接用户项目：TeamMind（沙箱执行）/ EP-002（供应链安全）/ dsh-pentest（安全测试）"
  - "活跃度最高：pushed 2026-09-22（昨日），13,920★ 持续增长"
  - "Benchmark 价值：沙箱隔离机制可对照 browser-use sandbox / OpenClaw sandbox deny / deepseek-harness 沙箱"
  - "品类轮换：09-21 记忆 → 09-22 computer-use → 09-23 沙箱隔离，避免连续同域边际递减"
  - "未考古（corpus 无 e2b 条目）；无需 refresh"
source_entry: "06_expansion_index/README.md [cand] E2B 沙箱 + APP（2026-09-01 首轮定向扫描，A 级）"
```
