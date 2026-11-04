# Job Manifest — ARCH-2026-11-04-001

- **run_id**: `ARCH-2026-11-04-001`
- **date**: 2026-11-04
- **domain focus**: Runtime（Agent 执行运行时 / 分布式运行时）第二周 Day3（11-02~11-08）
- **source**: 每日 Knowledge Archaeology Runtime（SOP）定时任务

## 候选排序（按 Knowledge Value / Agent-AI Engineering 相关性 / Novelty / 增长信号 / 近期发布 / 与已有 Corpus 连接 / 考古状态 / Benchmark 学习价值）

| # | 候选 | 等级 | 状态 | 排序理由 |
|---|------|------|------|----------|
| 1 | **durare**（SamuelXing/durare，Rust） | B（雷达 11-03 并入 durable 品类卡第 10 向） | **NEW → 选中** | DBOS 兼容 Rust durable-execution SDK：与已考古 **dbos-transact-py**（10-28 合入 main）构成同生态跨语言对照；与官方 dbos-transact-rust 存在显式 context vs task-local 设计对照（contrast 边）；62k 行 Rust 完整工程（macros/benchmarks/migrations/AGENTS.md/CHANGELOG 65KB）；11-01 仍活跃修复（#252 checkpoint 与 sends/stream writes 同提交）；durable 品类 Rust 语言栈补齐，Benchmark 学习价值高（同一语义跨语言实现差异） |
| 2 | durable-execution 品类卡维持观察 | B | OBSERVE | 9 向横向已完成（10-30），durare 为第 10 向并入；非单点考古对象 |
| 3 | PAP/Poppy（Personal Agent Protocol） | A（候选卡） | OBSERVE | 协议标准类、无代码仓库本体（规范 v0.1 10 月内交付承诺）；与已有 MCP/A2A 考古构成协议层对照，但非仓库考古对象，留待协议层专门考古 |
| 4 | semarun（PyPI 0.1.0） | B（雷达 11-03 观察位） | REJECTED | 恢复语义"第三派"候选（语义检查点），但仅 0.1.0 单版本、无公开代码仓库（PyPI 自述）、生态未明；不足构成深度考古对象 |
| 5 | multiagent-framework 选型卡 | B | OBSERVE | 格局综述类（LangGraph/CrewAI/AG2/MS Agent Framework），非单仓库；已有 langgraph/agent-framework 考古覆盖 |
| 6 | A2A 协议 | S5 | 已考古 | corpus main 已有 `a2a` 目录 |
| 7 | Agent Substrate | A（二号锚点） | 已考古 | corpus main 已有 `substrate`（10-26，ARCH-2026-10-26-001，commit 59a58a5） |
| 8 | restate / temporal / dapr-agents / dbos-transact-py / infinity | — | 已考古 | corpus 已含或 open PR 等待合入（#58~#63） |

## 选中项目

- **project**: durare
- **repository**: https://github.com/SamuelXing/durare.git
- **slug**: durare
- **mode**: initial
- **priority**: High（Runtime 领域 durable-execution 品类第 10 向；与 dbos-transact-py 跨语言对照；与官方 Rust SDK 设计对照）
- **reason**: ① 与已有 corpus（dbos-transact-py/restate/temporal/dapr-agents/substrate）连接最密，形成"同一语义跨语言实现差异"的 Benchmark 级对照；② Agent-AI Engineering 相关性高（durable execution = agent 长任务恢复的基础机制）；③ Novelty 高（2026-06 创建、v0.5.0 10-09 更新、crates.io downloads 1031）；④ 增长信号（11-01 仍在活跃 commit，30 open issues 含真实缺陷库）；⑤ 62k 行 Rust 完整工程（非浅层），含 AGENTS.md 治理文件 + 8k+ 行测试，适合完整考古
- **source_entry**: KnowlegeMap `03_expansion_queue/candidates/[cand]durable-execution-category-mainstreaming-2026-10.md` 第二周 Day2 增量（2026-11-03）：durare v0.5.0 = DBOS 兼容 Rust SDK，durable 品类第 10 向
- **evidence**: crates.io API（durare v0.5.0，repo=SamuelXing/durare，downloads 1031）；GitHub API（15★/2 forks/Apache-2.0/created 2026-06-10/pushed 2026-11-03）；浅克隆 HEAD `87ed2db`（fix(provider) #252，2026-11-01）；README 自述 "Rust SDK for DBOS durable execution, aligned by design with the DBOS Transact SDKs for Python, Go, TypeScript, and Rust"

## 不选理由（记录）

- **PAP**：协议/标准无代码仓库本体，规范 v0.1 未正式交付，不适合"Repository Snapshot + 代码考古"流程
- **semarun**：0.1.0 单版本 + 无仓库，观察位
- **品类卡**：横向对照已完成，非单点考古对象
- **Agent Substrate 复现**：KnowlegeMap 11-07 周五验证日任务，且 corpus 已考古（10-26）；SOP 考古与周五复现验证为两条独立线，不在今日重复考古
