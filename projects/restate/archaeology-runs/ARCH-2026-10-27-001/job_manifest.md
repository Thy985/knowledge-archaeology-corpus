# Job Manifest — ARCH-2026-10-27-001

## 选中项目
| 项 | 值 |
|---|---|
| job_id | ARCH-2026-10-27-001 |
| project | restate |
| repository | https://github.com/restatedev/restate.git |
| mode | initial |
| priority | P0 |
| commit 目标 | HEAD@selection（2026-10-26 pushed） |

## 候选排序（按 SOP 维度，非 star）
| 排序 | 项目 | 状态 | 维度依据 | 决策 |
|---|---|---|---|---|
| 1 | **Restate**（restatedev/restate，Rust） | NEW | **Knowledge Value 高**：durable execution 独立运行时核心机制（virtual objects/durable promises/journal segments）；**Agent-AI 相关性高**：agent 原生定位+Java/Kotlin SDK 是 TeamMind（Java 多 Agent 运行时）直接候选，对照 Dapr Agents Python；**Novelty 高**：corpus 无 restate；**增长信号**：Replit 采用 + v1.8.0/v1.8.1 10 月密集发布；**近期发布**：10-14/10-21/10-22 三 release；**Corpus 连接**：dapr-agents（durable execution Python 对照）/ substrate（快照-恢复对照）/ silver-shield（可审计执行链）；**Benchmark 学习价值**：virtual objects 编程模型、durable promises、幂等执行、日志段 | **选中**（P0） |
| 2 | Mastra（mastra-ai/mastra，TypeScript） | NEW | durable agents 原语化（1.78 scopes/queueSignal），10-01~17 15 minor release 增长强；但知识密度中等（编排 API 为主），Benchmark 价值低于独立运行时 | 备选 |
| 3 | Temporal（temporalio/temporal，Go） | 已知（品类卡已覆盖） | durable execution 鼻祖，但引擎数十万行 48h 难以精读；品类横向用 llm-info 文档即可 | 不选 |
| 4 | MS Agent Framework Durable Extension / AWS Lambda Durable Go SDK / Obelisk | NEW（品类卡） | 平台/文档型，无独立代码库考古价值；作为 7-way 横向素材周五验证日处理 | 不选（卡内素材） |

## 理由摘要
Restate 是 durable-execution-category-mainstreaming B 卡（10-27 建卡）中唯一"成熟开源运行时"且**与 TeamMind 直接连接**（04_connections/README.md:130 显式标注 "Restate Java/Kotlin SDK 直接候选，对照 Dapr Agents Python"）。昨天已考古 Agent Substrate（整机快照-恢复），今天考古 Restate（步骤持久化/durable execution 运行时）正好构成"恢复"两种哲学的互补证据线，服务周五 7-way 横向对照。4609★/228 forks（2026-10-26 pushed，v1.8.1）。

## 否决理由
- Mastra：框架层 API 封装为主，运行时内核机制（journal/state machine）非其主体
- Temporal：已作为品类基准被 B 卡覆盖；引擎规模超 48h 考古预算
- MS/AWS/Obelisk：平台文档/SDK 型，无独立权威代码库（MCP spec 类处理，归品类卡）

## source_entry
KnowlegeMap `03_expansion_queue/candidates/[cand]durable-execution-category-mainstreaming-2026-10.md`（B 级，10-27 建卡）+ `04_connections/README.md:130`（Restate→TeamMind durable execution 连接）+ `08_agent_centric/domains/agent-runtime.md` A 层缺口③（7-way 横向，周五验证日）
