# Job Manifest · ARCH-2026-10-10-001

## 候选排序（九维，不按 star）

| # | 候选 | 状态 | Knowledge Value | Agent-AI 相关 | Novelty | 增长信号 | 近期发布 | Corpus 连接 | 已考古 | 需 refresh | Benchmark 学习价值 | 决策 |
|---|------|------|-----------------|---------------|---------|----------|-----------|--------------|--------|-----------|---------------------|------|
| 1 | **dart_agent_core**（memex-lab/dart_agent_core，Dart，MIT） | **NEW** | A：Dart 侧 agent 运行时框架，corpus 全库无 Dart agent 框架（mcp/mem0/opencode/letta-code 均 Py/TS），Tafcm 生态直接复用对象 | 极高：完整 agentic loop + tool use + skills + sub-agent 委托 + planning + streaming + evals | 高：2.1.7 最新，2026-06 起活跃，10-09 仍有 push | 中：46★/14 forks，但开发活跃（pushed 2026-10-09）、pub.dev 生态位清晰 | 2.1.7（卡上 2.0.4 已过时） | 直接：agent 框架线（opencode/hermes-agent/mem0）+ evals 线（agentevals/evolver/ThinkingBox 评测设计） | 否 | N/A（initial） | 高：自带 agent evals + llm-evaluation topic，可直接对照 ThinkingBox 统计层/agentevals 评测面 | **选中 P0 initial** |
| 2 | **deepseek-harness**（deepseek-ai/deepseek-harness） | **REFRESH_REQUIRED** | A：0.1→0.2 主版本演进（v0.1.2-rc.1 已考古 → v0.2.1-alpha.2/master d743267，跨 11+ 版本，含 Claude Code Mods 桥接 10-01 新机制） | 极高 | 中（刷新对象，Mods 桥接为新增面） | 极高：v0.1.2→v0.2.1 连续 20+ tag | v0.2.1-alpha.1（10-03） | dsh 自身 corpus（09-11） | 是（ARCH-2026-09-11-001） | **是**：0.2 大版本 + Mods 兼容层（跨 harness 插件互通趋势） | 中（增量刷新） | 排序 2，留后续轮次 |
| 3 | AgentJudgeBench | NEW | A（论文级发现：judge 对齐随难度单调下降） | 极高（judge 评测=引擎 evaluation 层） | 高 | 中 | arXiv 2608.26623（08-27） | 引擎 evaluation 层/ThinkingBox | 否 | N/A | 高 | **排除**：无开源代码仓库（GitHub 搜索 0 结果，仅论文），不满足可考古代码仓库硬约束 |

## 选中项目

```yaml
job_id: ARCH-2026-10-10-001
project: dart_agent_core
repository: https://github.com/memex-lab/dart_agent_core.git
mode: initial
priority: P0
reason: >-
  九维排序第 1。Dart/Flutter 生态第一个可深度考古的 agent 运行时框架
  （完整 agentic loop + tool use + skills + sub-agent + planning + streaming + evals，
  多 provider LLM 支持，mobile-first local-first），与用户 Tafcm（Dart）项目直接连接；
  corpus 全库无 Dart agent 框架（知识空白补齐）；自带 agent evals（llm-evaluation topic），
  Benchmark 学习价值高——可与 ThinkingBox（Python 评测沙箱）/agentevals 对照评测设计；
  增长信号：2.1.7 最新版 + 2026-10-09 活跃 push。雷达来源：flutter-local-llm 卡 10-10 增量。
source_entry: >-
  KnowlegeMap 03_expansion_queue/candidates/ [cand]flutter-local-llm-2026.md
  （2026-10-10 雷达第 38 次增量：dart_agent_core 2.0.4；实测最新 2.1.7）
```

## 未选项目说明

- **dsh refresh**（排序 2）：刷新价值成立（0.2 主版本 + Mods 桥接），但按"每日最多 1 深度项目"本轮让位于全新对象 dart_agent_core；dsh 标记 REFRESH_REQUIRED 留后续轮次（v0.2.1 稳定后）。
- **AgentJudgeBench**（排序 3）：无开源代码仓库，不可考古；作为论文对象留给 Evaluation 周（10-12 起）精读，其 judge 校准结论可写入引擎 evaluation 层候选。
