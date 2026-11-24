# Daily Archaeology Job Manifest — 2026-11-25

- job_id: `KA-2026-11-25-001`
- run_id: `ARCH-2026-11-25-001`
- 生成时间: `2026-11-25T02:06:42+0800`
- 每日深度项目上限: 1
- 三仓库边界: KnowlegeMap 只读 / knowledge-archaeology-skill 只读 / knowledge-archaeology-corpus 可写（仅新增）

## 输入状态

- KnowlegeMap `origin/main`: `7eacfed5bc514b0a9254a3b01dda92e35ad8563c`（2026-11-25 00:34:15 +0800）
- 今日主题: `radar watch 2026-11-25: agent-desktop a11y desktop surface (A), GUI replay/effect-verification cluster (A), ComponentBench`
- 当前领域: Computer Use / Browser，第一周 Day3（只追增量）
- Corpus `origin/main`: `1b2b9fd7041295c884cc5f1c47dfe0ea7f4fd996`，63 个 projects；无 `agent-desktop`
- 未合并 PR: `#81 webmcp`、`#80 playwright-mcp`、`#79 browser-use refresh`
- 生产 Skill: `knowledge-archaeology` v3.2（本地）

## 候选排序

排序依据：Knowledge Value、Agent-AI Engineering 相关性、Novelty、增长信号、近期发布、与已有 Corpus 连接、是否已考古/需 refresh、Benchmark 学习价值；未按 star 排名。

| 排名 | 对象 | 状态 | 优先级 | 判断 |
|---|---|---|---|---|
| 1 | **agent-desktop**（`vercel-labs/agent-desktop`） | NEW | A | **选中**。桌面 CUA 的 a11y 表示层代表实现；ref 绑定快照身份 + 结构化错误 `disposition.retry` 把“动作回执”做成可验证契约；有真实代码、CLI/MCP 面、跨平台后端与并发/会话机制，适合做实现级考古。npm 最新 `1.3.1`（11-13），高于候选卡所记 1.2.0，候选卡日期/版本需在 Snapshot 中校正。 |
| 2 | GUI 回放/效果核验族（OpenAdapt / EchoPath / Activity Frames / AgentRR / agrepl） | NEW | A（品类） | 队列。知识价值高（“屏幕成功 ≠ 系统成功”，54/90→0/90），但由 5 篇论文/多个独立对象组成，不是单一仓库；本轮“1 项目”约束下不适合做仓库考古。留待周五验证或单独品类研究。 |
| 3 | browser-use | REFRESH_REQUIRED | A− | 队列。2026-11-22 已完成 REFRESH（PR #79 open），11-24/25 无新 release；同项目短期内重复会挤占每日唯一深度名额。 |
| 4 | playwright-mcp | REFRESH_REQUIRED | A− | 队列。2026-11-23 刚完成 initial（PR #80 open）；WebMCP `webmcp_<tool>` 工具面已成为新变量，但 1 天内重复同项目不符合多样性目标。 |
| 5 | webmcp | COMPLETED | B | 2026-11-24 完成 initial（PR #81 open，CG-DRAFT）；无新规范 commit，暂不 refresh。 |
| 6 | browser-harness | COMPLETED | B | `v0.1.13 @ afbcc38` 已考古；无新版本。 |
| 7 | webwright | COMPLETED | B | 上次 `bc26750`，当前 `35dbcf3`；未见重大版本/架构变化。 |
| 8 | ComponentBench | EVIDENCE_ONLY | B | 论文/评测证据，不是可克隆仓库；作为 Benchmark case 素材，不占深度考古名额。 |
| 9 | MAF PyPI supply-chain（1.26.1/1.26.2） | EVENT / OBSERVE | B+ | 事件已闭环处置，价值在制品供应链检测清单；无可考古的稳定发布仓库（相关 GitHub 仓库一度 404），本轮不选。 |

## 选中 Job

```yaml
job_id: KA-2026-11-25-001
project: agent-desktop
repository: https://github.com/vercel-labs/agent-desktop.git
mode: initial
priority: A
source_entry:
  - KnowlegeMap/07_radar_watch/current_focus.md (2026-11-25, Day3)
  - KnowlegeMap/05_scan_logs/2026-11/2026-11-25_radar-watch_scan.md
  - KnowlegeMap/03_expansion_queue/candidates/[cand]agent-desktop-a11y-desktop-2026-11.md
  - KnowlegeMap/08_agent_centric/domains/agent-computer-use.md (A/B/C/D/G 层)
```

## 选择理由

1. **Knowledge Value**：它不是“另一个桌面自动化工具”，而是把三个通常隐式的东西显式化——结构化的 UI 表示、绑定快照身份的 ref、可机器判断的重试安全性。正好落在能力树 A（环境表示）/ B（动作空间与 grounding）/ C（自验证）/ D（会话与状态）/ G（人机交接）的交叉点。
2. **Agent-AI Engineering 相关性**：CLI 无模型、循环留在 agent；MCP stdio 面把命令暴露为工具；headless 默认与 `--headed` 物理输入分界，直接关系 agent 的动作权威与人类确认点。
3. **Novelty**：`STALE_REF` / `AMBIGUOUS_TARGET` / `disposition.retry` 是把“目标失效/目标歧义/是否可重放”写进命令契约，而不是让 agent 从截图或自然语言错误里猜。
4. **增长信号 / 近期发布**：npm `1.3.1`（2026-11-13），48 个版本、约 5.0 万周下载（候选卡二手数据，包级待核）；仓库 HEAD 可访问。
5. **与 Corpus 连接**：Corpus 无 `agent-desktop`；可与 `webwright`（脚本化浏览器/可复现验证）、`browser-harness`（浏览器控制面）、`webmcp`（页面声明工具）形成“桌面 a11y vs 浏览器页面工具 vs 脚本回放”的对照。
6. **Benchmark 学习价值**：可用 ComponentBench 的 AX/Pixel 差异作为外部评测参照；仓库自身的错误契约可作为“失败可分类、可重试性可判定”的 Gold Record 素材。
7. **未选 GUI 回放族的原因**：知识密度更高，但不是单一仓库，且候选卡明确“未独立复现”；本轮要求 Repository Snapshot + 实现级证据，选一个可克隆、可逐行追溯的项目更符合任务定义。

## 预期产出

- Repository Snapshot：commit/branch/版本/时间戳 + 语言/入口/核心模块/数据结构/状态/测试/配置/权限治理/外部依赖。
- Archaeology Package：00–06 + `run_metadata.yaml`。
- Independent Validation：判定统计 + 3 成功 + 3 错误 + 遗漏 + Benchmark case。
- Corpus：`projects/agent-desktop/` + `archaeology-runs/ARCH-2026-11-25-001/`，独立 PR。
- Skill Evolution：仅在暴露系统性 skill 缺陷时创建。

## 边界

- 目标仓库只读；不修改 KnowlegeMap；不修改生产 Skill。
- 不覆盖 Corpus 历史；本 PR 仅新增 `projects/agent-desktop/`。
- 飞书默认不落盘（本轮无显式授权）。
- 所有事实必须可追溯到仓库实际内容；二手版本/下载量数据标 Observation/待核，不冒充 Fact。
