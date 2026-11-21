# 01 Project Layer · Data Model & Lifecycle

## 1. 数据模型（ER 要点）

```
organization (OrganizationMixin: 所有实体)
   └── project (ProjectMixin: block / passage 等)
        ├── block ──< block_history (sequence_number, current_history_entry_id 指针)
        │     ├── blocks_agents      (agent_id, block_id, block_label)  ← 多对多
        │     ├── blocks_conversations
        │     ├── groups_blocks
        │     ├── identities_blocks
        │     └── blocks_tags
        ├── passage ──< passage_tag
        ├── source ──< sources_agents
        ├── archive ──< archives_agents
        └── agent ──< messages / steps / runs / tools_agents
```

**Block 的归属是"多宿主"而非"单宿主"**：同一 block 可同时挂在 agent、conversation、group、identity 上（四个 pivot 表），因此它的 `read_only` / `hidden` 是块级属性，不是某个宿主的权限。

## 2. 版本化：两套并存

### 2.1 DB 快照（`block_manager.py`）
- `checkpoint_block_async`（`:842`）：读当前 block 与 `current_history_entry_id`，**删除比 current 更靠后的历史**（保证线性 undo/redo），插入 `sequence_number = current+1` 的新 `BlockHistory`，再更新指针。
- `undo_checkpoint_block`（`:952`）/ `redo_checkpoint_block`（`:1004`）：按 `sequence_number` 回放字段。
- 并发：Block ORM 有 `version` 列 + SQLAlchemy `version_id_col` 乐观锁（`orm/block.py:56-60`）。
- 审计：`BlockHistory.actor_type` / `actor_id` 记录是 agent 还是 user 触发。

### 2.2 Git 快照（`block_manager_git.py`）
- 类 `GitEnabledBlockManager(BlockManager)`（`:30`），注入 `MemfsClient`。
- `enable_git_memory_for_agent`（`:359`）：给 agent 打 tag、建 repo、把现有 blocks 作为初始提交；若 tag 存在但 repo 缺失 → 从当前 blocks 重建；若 repo 缺部分 block → **backfill**。
- `get_block_at_commit`（`:510`）、`get_block_history`（`:533`，支持按 `{label}.md` 过滤）、`sync_blocks_from_git`（`:564`）。
- 注释明确："Sync all blocks from git to PostgreSQL. Use this to rebuild the PostgreSQL cache from **git source of truth**."（`:565-568`）

**判断**：这不是两套等价实现，而是**权威源迁移的中间态**——DB 仍是读写热路径，但 git 被声明为 truth；两者之间没有事务性双写，只有 backfill / sync 两个方向的修复动作。

## 3. 三级记忆的生命周期

| 层 | 创建 | 修改 | 淘汰/压缩 | 删除 |
|---|---|---|---|---|
| Core（block） | agent 创建时绑定模板（human/persona）；`memory` 工具 create | `update_block_value` → `update_memory_if_changed_async` → `block` 行更新 + checkpoint | 无自动淘汰；`limit` 为字符上限（默认 `CORE_MEMORY_BLOCK_CHAR_LIMIT`） | `delete_block_async`（`:275`）级联 pivot |
| Recall（messages） | 每条消息写入 | 不改写原文；通过摘要生成"摘要消息" | `compact_messages` / sliding window 按 token 阈值驱逐并摘要 | 随 conversation 删除 |
| Archival（passage） | `insert_passage`（`:543`） | passage 一般不修改，只增删 | 检索时 top-k；非自动淘汰 | `delete_agent_passage_by_id_async`（`:767`） |

## 4. 摘要（Recall 压缩）

- `summarize_via_sliding_window`（`summarizer_sliding_window.py:99`）：按 `eviction_percentage` 计算 cutoff，保留尾部消息，把被驱逐区间交给 LLM 摘要；日志打印 `messages_kept`（`:218`）。
- `compact_messages`（`compact.py:135`）：统一入口，支持 `trigger_threshold` 校验（`:147`），并按 provider 选择 summarizer 模型（默认轻量模型，失败回退 agent 自身 LLM 配置，`:73-118`）。
- 策略枚举：`compact` / `self` / `sliding_window` / `all`（`services/summarizer/`）。

**边界**：摘要会**改写可回忆内容**——原始消息仍在 `messages` 表，但进入上下文的只有摘要 + 尾部；这是"recall 不等于全文回放"的实现事实。

## 5. 删除与级联

- `blocks_agents` 对 `block.id + block.label` 复合外键 `ondelete=CASCADE`（`orm/blocks_agents.py:12-19`），删 block 会解绑 agent。
- `block_history.block_id` 对 block `ondelete=CASCADE`（`orm/block_history.py:42`），**删 block 即删历史**——版本化不能阻止删除本身。
- `archives_agents` 支持 archival 在 agent 间共享（迁移 `74e860718e0d_add_archival_memory_sharing`）。

## 6. 生命周期中的关键缺口（事实，非评价）

1. 没有 TTL / 遗忘策略：core block 不自动过期，archival passage 不自动衰减。
2. 版本化只覆盖 block，不覆盖 passage / message；archival 与 recall 无历史快照。
3. git 与 Postgres 之间无事务边界，只有补偿式 sync/backfill。
4. 删除 block 会级联删历史，审计链在删除动作后不可恢复。
