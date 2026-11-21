# Job Manifest — ARCH-2026-10-11-001

## 候选排序（九维：Knowledge Value / Agent-AI 相关性 / Novelty / 增长信号 / 近期发布 / Corpus 连接 / 是否已考古 / 是否需 refresh / Benchmark 学习价值；不按 star）

| 排序 | 候选 | 状态 | 模式 | 优先级 | 理由 |
|---|---|---|---|---|---|
| **1** | **deepseek-harness**（deepseek-ai/deepseek-harness，master `d743267` = **dsh-v0.2.1-alpha.2**）| REFRESH_REQUIRED | **refresh** | **P0 · 选中** | corpus 记录 0.1.2-rc.1/`76fda729`（2026-09-03 考古），远端已演进至 v0.2.1-alpha.2（跨 0.1.3/0.1.5/0.1.6/0.1.7/0.2.0/0.2.1 共 20+ tags）——含 Claude Code Mods 兼容层（10-03）、模型×Harness 联合训练（V4.1-Flash）、异构 agent 编排等重大演进；10-11 雷达重大事件日 11 把 dsh 列为 Evaluation 周（10-12 起）验证对象（"验证计划需加入评测隔离性检查"）；本 run 是 skill 的 **refresh 模式首例**（版本演进对照考古，Benchmark 学习价值高）；与 corpus 连接强（dsh + dsh-memory-evolve + agentevals + dart-agent-core 评测线） |
| 2 | flutter_local_agent_kit（pub 1.0.2，offline-first RAG+ReAct+Material 3 UI）| NEW | initial | P1 | Dart 本地 agent 新生态（10-10 雷达增量），与 Tafcm 直接相关；但同类 Dart 框架（dart_agent_core）刚于 10-10 考古完成，边际学习价值低于 dsh refresh；留后续轮 |
| 3 | flutter_agentic（7 家 provider 统一 API + ReAct + on-device Gemma/GGUF）| NEW | initial | P2 | 同上，Dart agent 生态第三实例；作为 dart_agent_core 的横向对照可留待与 flutter_local_agent_kit 一并处理 |
| 4 | agentic-attack（事件跟踪卡）| QUEUED | — | P2 | G1 威胁范式事件链（非代码仓库），10-11 已追加重大事件日 11（Anthropic 评测隔离）；事件聚合属雷达职责，不适合仓库考古；其"评测隔离"素材由 dsh refresh 与 Evaluation 周消化 |
| 5 | agentjudge-bench | QUEUED | — | P3 | 上轮确认仅 arXiv 2608.26623 论文、无代码仓库 → 排除（论文精读另类任务）|

**其他未考古候选**（30+ 张卡）维持观察：a2a-protocol、agent-firewall、agent-formal-verification、agent-harness-control-plane、agentic-benchmarks、agentic-graphrag、agentic-inference-infra、evals-framework-landscape、haaf-trustworthy-agent-eval、hermes-agent（已考古）、otel-genai-observability、trace-based-agent-eval、promptfoo-agent-evals、local-edge-llm、e2b-sandbox、webwright（已考古）等——多数为 Evaluation 周（10-12 起）候选素材，本周内按周计划逐日消化。

## 选中项目

- **job_id**: ARCH-2026-10-11-001
- **project**: deepseek-harness
- **repository**: https://github.com/deepseek-ai/deepseek-harness.git
- **mode**: refresh
- **priority**: P0
- **reason**: 版本 0.1.2-rc.1 → 0.2.1-alpha.2 的重大演进 + 10-11 雷达将其列为 Evaluation 周验证对象 + skill refresh 模式首例 Benchmark 价值 + corpus 强连接
- **source_entry**: KnowlegeMap 03_expansion_queue/validated/[val]deepseek-harness-2026.md（雷达增量 2026-10-09：v0.2.1-alpha.1 Claude Code Mods 兼容层）；git ls-remote 实测 master=`d743267`（= dsh-v0.2.1-alpha.2）

## 考古范围界定（refresh）

- **基线**：corpus `projects/deepseek-harness/`（2026-09-03 考古，run ARCH-2026-09-03-001，0.1.2-rc.1/`76fda729`，38 Facts/32 EK/8 KO/9 Candidates）
- **本 run 目标**：以新 commit `d743267` 全量重读核心面，产出**更新至 0.2.1-alpha.2 的完整 Package**（00-06 + run_metadata），显式标注与 0.1.2-rc.1 基线相比的**演进差异**（新增/变更/移除模块、机制、配置），保留原考古中仍成立的结论（去重不覆盖历史 run，仅新增本 run 目录）
- **重点关注**：Claude Code Mods 兼容层（插件 API 子集验证）、模型×Harness 联合训练痕迹、动态系统提示词与 KV Cache 保留、评测隔离相关（对照 10-11 雷达 fail-closed 网络要求）、0.2 阶段的架构承诺维持性（"无特权内核"）
