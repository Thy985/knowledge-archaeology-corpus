# Flow Atlas（七类流 · 28 条边）

> 每条边给出 symbol / file / condition / state transition，均可回溯到仓库代码。运行态验证 = 0。

## Control（控制流）

| ID | From → To | 触发/条件 | 证据 |
|---|---|---|---|
| C-01 | agent loop → tool_executor | 模型产出 tool call，按 tool name 分派到 `core_tool_executor.execute` | `agents/letta_agent_v3.py`；`services/tool_executor/` |
| C-02 | tool_executor.memory_replace → agent_state.memory | `update_block_value(label, new_value)` | `core_tool_executor.py:395` |
| C-03 | memory 变更 → `update_memory_if_changed_async` | 工具方法末尾显式调用 | `core_tool_executor.py:398` |
| C-04 | `update_memory_if_changed_async` → `block_manager.update_block_async` | 逐 label diff 且值变化 | `agent_manager.py:1770-1781` |
| C-05 | `update_block_async` → `checkpoint_block_async`（由调用方触发） | 需要留历史时 | `block_manager.py:211/842` |

**例外/旁路**：`memory_rethink`（legacy）只改 `agent_state.memory`，**不调用** `update_memory_if_changed_async`（`function_sets/base.py:488-518`）→ 若走该路径，改动是否落库取决于外层循环是否统一 flush。

## State（状态流）

| ID | From → To | 状态迁移 | 证据 |
|---|---|---|---|
| S-01 | `Memory`（in-context）→ `Block` 值 | `update_block_value` 直接改 Pydantic 对象 | `schemas/memory.py:771-781` |
| S-02 | `Block` row → `BlockHistory` row | checkpoint：`sequence_number = current+1`，先删未来项 | `block_manager.py:842-912` |
| S-03 | `current_history_entry_id` 指针 | undo/redo 移动指针 → 回放字段 | `block_manager.py:952-1048` |
| S-04 | `Block.version` 乐观锁 | 每次更新 version+1；冲突抛 StaleDataError | `orm/block.py:56-60` |

**并发边界**：`version_id_col` 只保护单行更新；checkpoint 与 block 更新不在同一事务语义内（checkpoint 自己开 session）。

## Data（数据流）

| ID | From → To | 变换 | 证据 |
|---|---|---|---|
| D-01 | `Block.value` → system prompt | `Memory.compile()` 拼接 blocks（standard/git/line-numbered） | `memory.py:688-745` |
| D-02 | `Passage.text` → embedding | `insert_passage` 调 embedding client | `passage_manager.py:570-590` |
| D-03 | embedding → pgvector / Turbopuffer | `archive.vector_db_provider == TPUF` 时双写 | `passage_manager.py:612-628` |
| D-04 | `messages` → summary | sliding window 摘要，替换被驱逐区间 | `summarizer_sliding_window.py:99-218` |

**数据丢失点**：D-03 的 TPUF 失败被非 strict 模式吞掉；D-04 的原文不再进入上下文。

## Evidence（证据/审计流）

| ID | From → To | 记录内容 | 证据 |
|---|---|---|---|
| V-01 | 写操作 → `BlockHistory` | `actor_type`（LETTA_AGENT / LETTA_USER）+ `actor_id` | `block_manager.py:885-892` |
| V-02 | block 创建/更新 → `created_by_id` / `last_updated_by_id` | 用户身份 | `schemas/block.py:73-74` |
| V-03 | git 模式 → commit | `{label}.md` + commit message（enable/backfill/update） | `block_manager_git.py:359-484` |

**审计缺口**：删除 block 级联删 history（E-34）；passage/message 无历史。

## Authority（权限流）

| ID | From → To | 判定 | 证据 |
|---|---|---|---|
| A-01 | HTTP 请求 → user | password 相等或 `api_key_to_user` | `auth_token.py:11-17` |
| A-02 | user → org/project scope | ORM 查询带 `organization_id`/`project_id` | `orm/block.py:22` |
| A-03 | memory 编辑 → read_only 判定 | `agent_state.memory.get_block(label).read_only` | `core_tool_executor.py:354` |
| A-04 | 共享 block → 多 agent | `blocks_agents` 多对多，无 per-agent 权限列 | `orm/blocks_agents.py:7-30` |

**边界**：A-03 只在新 executor 路径生效；legacy `core_memory_append/replace` 无该判定。

## Memory（记忆流）

| ID | 层 | 写入 | 读取 | 证据 |
|---|---|---|---|---|
| M-01 | Core | tool → memory 对象 → DB block | 每次 compile 进 prompt | `memory.py:688` |
| M-02 | Recall | message 追加 | 上下文消息列表 + 摘要 | `summarizer/compact.py` |
| M-03 | Archival | `insert_passage` | 向量检索 top-k | `passage_manager.py:543` |
| M-04 | Core 共享 | block 挂到多 agent/conversation/group/identity | 各宿主各自 compile | `orm/blocks_agents.py` 等 |
| M-05 | Core 版本 | BlockHistory / git | undo/redo、get_block_at_commit | `block_manager.py:952`；`block_manager_git.py:510` |

## Policy（策略流）

| ID | 策略 | 生效点 | 证据 |
|---|---|---|---|
| P-01 | 仓库 deprecated：新功能应去 letta-code | 文档层，无代码强制 | `AGENTS.md:1-16` |
| P-02 | AI 贡献须披露、人类复核 | 文档层 | `AGENTS.md:18-24`；`AI_POLICY.md` |
| P-03 | 漏洞仅私下上报，7 天内给修复时间线 | 文档层 | `SECURITY.md:3-10` |

**Policy 特征**：三条都是"文档声明"，没有对应的代码检查/CI 强制（例如无 workflow 校验 AI 披露）。治理闭环在文档处中断。
