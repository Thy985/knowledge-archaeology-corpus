# Engineering Knowledge（20 条 · EK Graph 节点）

> 层级：L1/L2（宽底座）。每条声明 `type / statement / evidence / links / epistemic / value`。全部 S3（代码可读，未运行）。

## EK-01 · Memory 对象只承载 Core，不承载 Recall/Archival
- **type**: implementation
- **statement**: `Memory` 的字段只有 `blocks`、`file_blocks`、`git_enabled`；Recall（messages）与 Archival（passage）不在该对象内。三级记忆是三个子系统，不是一个类的三个字段。
- **evidence**: E-04, E-05
- **links**: `subsystem` → EK-02；`dependency` → EK-07
- **value**: B

## EK-02 · 三级记忆的差异是"准入方式"而非"存储介质"
- **type**: mechanism
- **statement**: Core 每次 compile 都进入 system prompt；Recall 以消息缓冲存在，超阈值后被摘要压缩；Archival 只有被检索时才进入上下文。三者的成本结构不同（常驻 token / 压缩 LLM 调用 / 向量检索）。
- **evidence**: E-04, E-27, E-29
- **links**: `mechanism` → EK-16；`subsystem` → EK-14
- **value**: A

## EK-03 · Block 是多宿主共享对象
- **type**: implementation
- **statement**: 同一 block 可通过 `blocks_agents` / `blocks_conversations` / `groups_blocks` / `identities_blocks` 同时挂到多个宿主；`(agent_id, block_label)` 唯一，保证一个 agent 内 label 不重复。
- **evidence**: E-08, E-10, E-11
- **links**: `subsystem` → EK-19
- **value**: B

## EK-04 · BlockHistory 是线性 undo/redo 栈
- **type**: mechanism
- **statement**: 每次 checkpoint 先删除 `sequence_number` 大于当前的所有历史，再插入 `current+1` 的快照并更新 `current_history_entry_id`。undo/redo 通过移动指针对应的序号回放字段，不新增分支。
- **evidence**: E-09, E-12, E-13
- **links**: `mechanism` → EK-05；`constraint` ← EK-20
- **value**: A

## EK-05 · git 被声明为 source of truth，Postgres 降为缓存
- **type**: decision
- **statement**: `GitEnabledBlockManager` 维护每 agent 的 git repo（`{label}.md`），提供按 commit 读取与历史查询；`sync_blocks_from_git` 的 docstring 明说"rebuild the PostgreSQL cache from git source of truth"。
- **evidence**: E-14, E-16, E-17
- **links**: `contrast` → EK-04（同一问题的两代解法）
- **value**: A

## EK-06 · git 与 Postgres 没有跨存储事务（且两种模式写入顺序不同）
- **type**: failure_path
- **statement**: 非 git 模式直接写 Postgres；git 模式则 **先 commit 到 git、再同步 Postgres 缓存**（`update_block_async` 的步骤 1/2）。两条路径都没有跨存储事务：git 模式若 PG 同步失败，缓存落后于真源；非 git 模式无 git 副本。修复依赖 `backfill`（缺 block）与 `sync_blocks_from_git`（整库重建），无冲突解决策略。
- **evidence**: E-15, E-16, E-17
- **links**: `dependency` → EK-05
- **value**: A

## EK-07 · 记忆写入走"状态对象 → 服务端 diff → DB"
- **type**: mechanism
- **statement**: 工具执行器改 `agent_state.memory`，随后 `update_memory_if_changed_async` 重新 compile、与旧 system prompt 比较、逐 label diff，只把变化的 block 写回 DB。
- **evidence**: E-20, E-25, E-26
- **links**: `causal` → EK-08；`causal` → EK-09
- **value**: A

## EK-08 · 变更检测以"渲染后的提示词"为基准
- **type**: failure_path
- **statement**: 是否写库取决于 `new_memory_str not in system_message.content[0].text`。这使持久化逻辑耦合渲染格式：渲染器变更、或相同文本恰好出现在 prompt 其他区段，都可能改变写判定。
- **evidence**: E-25
- **links**: `dependency` → EK-09
- **value**: A

## EK-09 · `read_only` 的执行点在工具执行器
- **type**: security_boundary
- **statement**: 至少 9 处 `read_only` 检查位于 `core_tool_executor` 的记忆编辑方法中；`Memory` / `Block` 对象本身不拒绝写入。
- **evidence**: E-20, E-33
- **links**: `contrast` → EK-10；`dependency` ← EK-08
- **value**: A

## EK-10 · legacy 记忆函数没有 `read_only` 校验
- **type**: failure_path
- **statement**: `core_memory_append` / `core_memory_replace` 直接在函数体内改 `agent_state.memory`，无 read_only、无长度上限、无审计参数。若该工具路径被注册，只读约束不生效。
- **evidence**: E-18, E-19
- **links**: `mechanism` → EK-09（同一约束的两种实现层）；`contrast` → EK-09
- **value**: A

## EK-11 · Schema 函数是"声明"，真实实现在 executor
- **type**: boundary
- **statement**: `memory()`、`memory_replace`、`memory_apply_patch` 的模块级函数直接抛 `NotImplementedError("This should never be invoked directly.")`；docstring 描述语义，实际行为在 `core_tool_executor`。
- **evidence**: E-24
- **links**: `constraint` → EK-09
- **value**: B

## EK-12 · `memory_apply_patch` 支持跨 block 的 unified diff
- **type**: implementation
- **statement**: 支持 `*** Add Block` / `*** Delete Block` / `*** Update Block` / `*** Move to`，可一次修改多个 block；无 `***` 头时退化为单 block patch。
- **evidence**: E-22
- **links**: `mechanism` → EK-13
- **value**: B

## EK-13 · 精确编辑要求唯一匹配且禁止行号前缀
- **type**: constraint
- **statement**: `memory_replace` 对 old_string 出现 0 次或 >1 次都报错并列出命中行；`memory_rethink` 拒绝 `\nLine N:` 前缀与行号警告横幅。行号仅为展示，不得回写。
- **evidence**: E-21, E-23
- **links**: `constraint` → EK-11
- **value**: B

## EK-14 · archival 写入是"SQL 先写 + 可选向量双写"
- **type**: implementation
- **statement**: `insert_passage` 总是写 SQL；仅当 `archive.vector_db_provider == TPUF` 时再写 Turbopuffer，并复用同一批 embedding 与 passage id。
- **evidence**: E-27
- **links**: `subsystem` → EK-02
- **value**: B

## EK-15 · 向量双写失败在非 strict 模式被吞掉
- **type**: failure_path
- **statement**: TPUF 写入异常在 `strict_mode=False` 时只 `logger.error`，SQL 侧已提交，检索可能返回"库里有、向量里没有"的不一致视图。
- **evidence**: E-27
- **links**: `causal` → EK-14
- **value**: A

## EK-16 · Recall 压缩按比例驱逐并摘要
- **type**: mechanism
- **statement**: sliding window 以 `eviction_percentage` 计算 cutoff，保留尾部消息，把被驱逐区间摘要；`compact_messages` 额外支持 `trigger_threshold` 校验压缩后是否达标。
- **evidence**: E-29, E-30
- **links**: `mechanism` → EK-17；`subsystem` → EK-02
- **value**: A

## EK-17 · 摘要不等于原文回放
- **type**: boundary
- **statement**: 被驱逐消息不再进入上下文，模型只能看到摘要与尾部；原始消息仍在 `messages` 表，但没有按原文回放的工具路径（除显式历史查询）。
- **evidence**: E-29
- **links**: `contrast` → EK-01
- **value**: A

## EK-18 · 租户隔离靠 ORM 过滤，不是 schema 隔离
- **type**: security_boundary
- **statement**: 所有实体继承 `OrganizationMixin`；block/passage 另有 `project_id`。隔离由查询条件（`actor.organization_id`）保证，多租户共用同一套表。
- **evidence**: E-32
- **links**: `constraint` → EK-19
- **value**: A

## EK-19 · 共享 block 的 `read_only` 无法表达 per-agent 权限
- **type**: design_gap
- **statement**: `read_only` 是 block 字段（注释："permissions of the agent"），而 block 可被多 agent 共享。同一 block 无法同时满足"A 可写、B 只读"。
- **evidence**: E-10, E-33
- **links**: `subsystem` → EK-03；`contrast` → EK-18
- **value**: A

## EK-20 · 删除 block 会级联删除其历史
- **type**: boundary
- **statement**: `block_history.block_id` 外键为 `ondelete=CASCADE`。删除动作使版本化与审计链在该 block 上不可恢复。
- **evidence**: E-34
- **links**: `constraint` → EK-04
- **value**: B


## EK-21 · 同一私有方法被重复定义
- **type**: failure_path
- **statement**: `BlockManager._move_block_to_sequence` 在 `block_manager.py:760` 与 `:914` 各定义一次；Python 语义下后者覆盖前者。两处签名相同，行为是否等价未核实，但这是可观察的实现风险（潜在死代码或版本合并残留）。
- **evidence**: IF-12（`grep -n "def _move_block_to_sequence"`）
- **links**: `constraint` → EK-04（影响历史回放的实现一致性）
- **value**: B

## EK-22 · git→PG 缓存同步漏传 read_only / metadata
- **type**: failure_path
- **statement**: `_sync_block_to_postgres` 接受 `read_only` 与 `metadata` 参数，但 `update_block_async` 的调用只传 `value/description/limit`。在 git 模式下修改这两个字段不会回写 PG 缓存，缓存与真源可产生字段级不一致。
- **evidence**: `block_manager_git.py:232-244`（调用）vs `:82-92`（签名）
- **links**: `causal` → EK-06；`dependency` → EK-05
- **value**: A

## EK-23 · legacy `rethink_memory` 可创建新 block
- **type**: boundary
- **statement**: 当 `target_block_label` 不存在时，`rethink_memory` 直接构造 `Block` 并 `set_block`，即 legacy 路径不仅能改值，还能新增 core memory 块，且同样没有 `read_only` 校验。
- **evidence**: `base.py:283-310`
- **links**: `mechanism` → EK-10；`contrast` → EK-11
- **value**: B
