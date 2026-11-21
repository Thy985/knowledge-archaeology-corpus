# Evidence Index（40 条 Fact）

| ID | Fact | 来源（file:line） | 强度 |
|---|---|---|---|
| E-01 | 对象为 archive 分支 legacy V1 server，commit `56ba9c25`，2026-08-13 | `git log -1`（snapshot） | S3 |
| E-02 | 规模：1156 文件 / 878 py / 248,417 行 / tests 166 / test_* 95 / 31 workflows | `os.walk` 脚本枚举（排除 `.git`） | S3 |
| E-03 | 版本 `letta 0.16.8`，Python `>=3.11,<3.14`，Apache-2.0 | `pyproject.toml:1-12` | S3 |
| E-04 | `Memory` = in-context core memory，字段 `blocks/file_blocks/git_enabled` | `schemas/memory.py:68-80` | S3 |
| E-05 | `Memory.compile()` 三渲染分支：git / line-numbered / standard；react 不渲染 | `schemas/memory.py:688-745` | S3 |
| E-06 | `Block` schema 含 `label/value/limit/read_only/hidden/project_id` | `schemas/block.py:13-104` | S3 |
| E-07 | `BlockResponse.read_only` 标 `deprecated=True` | `schemas/block.py:103` | S3 |
| E-08 | Block ORM 有 `current_history_entry_id` 与 `version` 乐观锁 | `orm/block.py:50-60` | S3 |
| E-09 | `BlockHistory` 保存快照 + `sequence_number` + `actor_type/actor_id` | `orm/block_history.py:10-48` | S3 |
| E-10 | `blocks_agents` 多对多，唯一约束 `(agent_id, block_label)` 与 `(agent_id, block_id)` | `orm/blocks_agents.py:7-30` | S3 |
| E-11 | Block 可同时挂 agent/conversation/group/identity（四个 pivot） | `orm/blocks_conversations.py`, `groups_blocks.py`, `identities_blocks.py` | S3 |
| E-12 | `checkpoint_block_async` 截断未来历史、生成下一序号、更新指针 | `block_manager.py:842-912` | S3 |
| E-13 | `undo_checkpoint_block` / `redo_checkpoint_block` 按序号回放 | `block_manager.py:952-1048` | S3 |
| E-14 | `GitEnabledBlockManager(BlockManager)` 注入 MemfsClient | `block_manager_git.py:30-42` | S3 |
| E-15 | `enable_git_memory_for_agent` 支持 tag 存在但 repo 缺失时重建、缺 block 时 backfill | `block_manager_git.py:359-484` | S3 |
| E-16 | `get_block_at_commit` / `get_block_history` / `sync_blocks_from_git` | `block_manager_git.py:510-596` | S3 |
| E-17 | 注释声明 "rebuild the PostgreSQL cache from git source of truth" | `block_manager_git.py:565-568` | S3 |
| E-18 | legacy `core_memory_append` 直接拼接并 `update_block_value`，无校验 | `functions/function_sets/base.py:246-261` | S3 |
| E-19 | legacy `core_memory_replace` 仅校验 old_content 存在，无 read_only 检查 | `base.py:263-281` | S3 |
| E-20 | v2 `memory_replace` 在 executor 中先检查 `read_only` | `services/tool_executor/core_tool_executor.py:346-356` | S3 |
| E-21 | `memory_replace` 要求 old_string 唯一（0/多次均报错） | `core_tool_executor.py:375-390` | S3 |
| E-22 | `memory_apply_patch` 支持多 block（Add/Delete/Update/Move） | `core_tool_executor.py:403-470`; `base.py:453-486` | S3 |
| E-23 | `memory_rethink` 拒绝行号前缀与警告横幅 | `base.py:488-518` | S3 |
| E-24 | `memory()`、`memory_replace`、`memory_apply_patch` 的 schema 函数直调会 `NotImplementedError` | `base.py:10/71/311/453` | S3 |
| E-25 | `update_memory_if_changed_async` 以 compiled prompt 字符串判断是否有变化 | `agent_manager.py:1762-1770` | S3 |
| E-26 | 仅对值变化的 block 调 `update_block_async`；新增 block 依赖创建方法已持久化 | `agent_manager.py:1770-1785` | S3 |
| E-27 | `insert_passage` 先写 SQL，再在 TPUF 时双写；非 strict 模式失败仅记日志 | `passage_manager.py:543-640` | S3 |
| E-28 | archival 检索按 `archive_id` 与 embedding；archive 可为 agent 默认归档 | `passage_manager.py` + `archive_manager` | S3 |
| E-29 | `summarize_via_sliding_window` 按 eviction 比例切分，保留尾部 | `summarizer_sliding_window.py:99-218` | S3 |
| E-30 | `compact_messages` 支持 `trigger_threshold` 校验；summarizer 模型失败回退 | `summarizer/compact.py:42-180` | S3 |
| E-31 | 鉴权 `get_current_user`：password 或 api_key → user | `server/rest_api/auth_token.py:11-17` | S3 |
| E-32 | 实体继承 `OrganizationMixin`；Block/Passage 另有 `ProjectMixin` | `orm/block.py:22` | S3 |
| E-33 | `read_only` 注释为 "permissions of the agent"，字段位于 block | `orm/block.py:48-49` | S3 |
| E-34 | `block_history.block_id` 外键 `ondelete=CASCADE`（删 block 删历史） | `orm/block_history.py:42` | S3 |
| E-35 | `routers/v1` 30 个文件，含 blocks/agents/passages/archives/identities/groups | `server/rest_api/routers/v1/` | S3 |
| E-36 | 测试覆盖记忆：`test_memory.py` / `test_block_manager_noop_update.py` / `test_compaction_thresholds.py` / `integration_test_summarizer.py` / git sync 测试 | `tests/` | S3 |
| E-37 | 31 个 CI workflow（unit/integration/migration/docker/多本地模型） | `.github/workflows/` | S3 |
| E-38 | AGENTS.md 声明仓库 deprecated，活动开发在 letta-code | `AGENTS.md:1-16` | S3 |
| E-39 | SECURITY.md 仅漏洞上报流程，无威胁模型 | `SECURITY.md:1-10` | S3 |
| E-40 | 167 个 Alembic 迁移，含 `refactor_agent_memory`、`add_block_history_tables`、`add_archival_memory_sharing`、`add_read_only_column` | `alembic/versions/` | S3 |

> 全部为 **S3（已实现，代码可读）**；无 S4/S5：测试未运行，服务未启动。
