# 01 Project Layer · Architecture

## 1. 运行形态

三代实现共存于同一仓库：

| 代 | 入口 | 说明 |
|---|---|---|
| CLI/本地 | `letta`（`pyproject.toml` scripts） | Typer + questionary |
| HTTP 服务 | `letta/server/rest_api/app.py`（FastAPI） | 30 个 v1 router（agents/blocks/passages/sources/archives/identities/groups/...） |
| 执行循环 | `letta/agents/base_agent.py` / `base_agent_v2.py` / `letta_agent.py` / `letta_agent_v2.py` / `letta_agent_v3.py` | v1 → v2 → v3 并存；v3 2134 行 |

`agents/` 共 14 文件 / 8,050 行，包含 `letta_agent_batch.py`（批处理）、`ephemeral_agent.py` / `ephemeral_summary_agent.py`（临时 agent）、`voice_agent.py` / `voice_sleeptime_agent.py`。

## 2. 分层

```
HTTP router (letta/server/rest_api/routers/v1, 30 个)
   │  FastAPI + Pydantic request/response
   ▼
managers/（22 文件 / 22,928 行，编排层）
   │
   ▼
services/（119 文件 / 41,334 行，业务层）
   ├── agent_manager.py        3,599 行（agent 状态、memory 持久化、archival 检索）
   ├── block_manager.py        1,048 行（core memory block CRUD + checkpoint）
   ├── block_manager_git.py      596 行（git 版本化子类）
   ├── passage_manager.py      1,103 行（archival 写入/检索）
   └── summarizer/（compact / sliding / self / thresholds）
   │
   ▼
orm/（SQLAlchemy 2.x async；Block / BlockHistory / BlocksAgents / Passage / Source / Identity）
   │
   ▼
PostgreSQL（默认 + pgvector）／可选 Turbopuffer（archival 双写）
```

## 3. 核心抽象

| 抽象 | 定义位置 | 语义 |
|---|---|---|
| `Memory` | `schemas/memory.py:68` | in-context 的 core memory：`blocks` + `file_blocks` + `git_enabled`；`compile()` 渲染为系统提示词片段 |
| `Block` | `schemas/block.py:67` / `orm/block.py:22` | 一段可编辑的上下文；字段含 `label/value/limit/read_only/hidden/project_id`、`current_history_entry_id`、`version`（乐观锁） |
| `BlockHistory` | `orm/block_history.py:10` | block 的单次历史快照：`sequence_number` + 全套字段 + `actor_type/actor_id` |
| `BlocksAgents` | `orm/blocks_agents.py:7` | agent ↔ block 多对多；唯一约束 `(agent_id, block_label)`、`(agent_id, block_id)` |
| `Passage` | `orm/passage.py` | archival 的最小单位：`text/embedding/embedding_config/archive_id/tags` |
| `AgentState` | `schemas/agent.py` | 一次请求的完整状态：`memory`、`messages`、`tools`、`tool_rules` |

## 4. 记忆的编译（Core → Prompt）

`Memory.compile()`（`memory.py:688-745`）按三条分支渲染：
1. `git_enabled=True` → `_render_memory_blocks_git()`（`memory.py:205`）；
2. `agent_type` ∈ {`sleeptime_agent`, `memgpt_v2_agent`, `letta_v1_agent`} 且模型为 Anthropic → 行号渲染 `_render_memory_blocks_line_numbered()`（`:175`）；
3. 其他 → 标准渲染 `_render_memory_blocks_standard()`（`:143`）。

React / workflow agent 不渲染 memory blocks（`:705` 的 `if not is_react`）。`available_skills` 明确为 request-scoped，不持久化进 compiled prompt（`:706-708` 注释）。

## 5. 写路径（一次记忆修改）

```
agent tool call
  → tool_executor（memory_replace / memory_insert / memory_apply_patch / memory_rethink）
  → agent_state.memory.update_block_value(label, value)
  → agent_manager.update_memory_if_changed_async(agent_id, new_memory)
       ├─ new_memory.compile(...) 与 system_message.content[0].text 比较
       ├─ 逐 label diff，仅对变化 block 调 block_manager.update_block_async
       └─ 从 DB 重新加载 blocks，回写 agent_state.memory
  → （可选）block_manager.checkpoint_block_async 写 block_history
  → （git 模式）GitEnabledBlockManager 写 repo 并回写 Postgres 缓存
```

关键：变更检测以 **编译后的系统提示词字符串** 为基准（`agent_manager.py:1768`），不是结构化 diff。

## 6. 多后端

- 默认：Postgres + pgvector（`archive.vector_db_provider`）。
- 可选：Turbopuffer（`insert_passage` 中 SQL 先写，再双写 TPUF；`strict_mode=False` 时双写失败只记日志——`passage_manager.py:620-628`）。
- LLM 侧：`llm_api/`（27 文件 / 9,494 行）+ `local_llm/`（34 文件 / 5,471 行）；CI 有独立 ollama / vllm / lmstudio workflow。
