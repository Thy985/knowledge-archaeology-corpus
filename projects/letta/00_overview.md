# 00 Overview — Letta legacy V1 API server

> run `ARCH-2026-11-09-001` · mode initial · branch `archive` · commit `56ba9c25` · skill v3.2

## 一句话定位

Letta（前身 MemGPT）的**上一代自托管服务端**：把"有状态 agent 的记忆"实现为 **Postgres 关系模型**——Core memory 是 `block` 表里被编译进系统提示词的文本块，Recall 是 `messages` 表加 LLM 摘要压缩，Archival 是 `passage` 表加向量检索；记忆修改走工具调用 → 服务端持久化 → `block_history` 快照，后期又叠加了"每 agent 一个 git 仓库"的版本化层。

## 为什么值得考古（与已有 corpus 的连接）

| 已有项目 | 记忆路线 |
|---|---|
| `mem0` | 抽取式、向量优先、**去图** |
| `graphiti` | 时序知识图谱、双时间事实 |
| `letta-code` | **文件即记忆**：每 agent 一个 git 仓库 + Markdown（2026 现行实现） |
| **本项目（letta archive）** | **数据库即记忆**：ORM + 三级记忆 + 服务端共享 block（上一代实现） |

`letta-code` 的考古已记录"main 分支归档为文档仓、旧 V1 在 archive 分支"。本次把 archive 分支打开，回答一个此前无法回答的问题：**同一产品为什么会从"数据库记忆"迁移到"文件记忆"**，以及迁移过程中哪些约束被保留、哪些被放弃。

## 三条核心结论

1. **三级记忆不是三个数据库，而是三种"进入上下文的方式"**：Core 常驻（每次编译进 system prompt）、Recall 可压缩（消息缓冲 + 摘要）、Archival 按需检索（passage + 向量）。三者共享同一套 `organization_id/project_id` 作用域，但生命周期与信任假设不同。
2. **版本化经历了两代：DB 线性 undo/redo → git 作为 source of truth**。`BlockHistory` 用 `sequence_number` + `current_history_entry_id` 做线性快照；`GitEnabledBlockManager` 之后把每个 agent 的 block 写成 `{label}.md`，并提供 `get_block_at_commit`、`get_block_history`、`sync_blocks_from_git`，注释明确 "rebuild the PostgreSQL cache from git source of truth"。
3. **写入门禁在工具执行器，不在记忆对象**。`memory_replace` 等 v2/v3 工具在 `core_tool_executor` 里检查 `read_only`；而 legacy `core_memory_append/replace` 直接改 `agent_state.memory`，函数体内**没有任何 read_only 或长度校验**——防护是否生效取决于"哪条工具路径被注册"。

## 质量与边界

| 项 | 值 |
|---|---|
| Facts / EK / KO / Candidates / Flow Edge | 40 / 20 / 6 / 6 / 26 |
| 代码精读 | ≈12,000 / 248,417 行（≈4.8%） |
| 运行验证 | **0**（无 Postgres/pgvector/Turbopuffer/LLM key） |
| 最高认知层级 | **L3 Pattern**（0 Principle） |

**必须声明的边界**
- 本仓库是 **deprecated legacy server**（AGENTS.md 原文），结论只描述"这一代实现如何工作"，不代表当前 Letta 运行时。
- `--depth 1` 浅克隆，无 git 历史，无法回溯修复史与设计演进顺序；所有"演进"判断来自**代码内并存的多代实现**。
- 未覆盖：`letta/managers/` 22.9k 行、30 个 router 全量、`llm_api` provider 实现、167 个迁移全文。
