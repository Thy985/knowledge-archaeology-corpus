# 01.2 · Data Model & Lifecycle

## 1. Chat：L0–L7 分层

| 层 | 枚举 | 写入者 | 召回 | 备注 |
|---|---|---|---|---|
| L0 `BASIC_INFO` | `l0_basic_info` | 基础画像 agent | **独立隐藏通道**，上限 10 | 不与其他层混排（`reader_base.py:136-139`） |
| L1 `RAW` | `l1_raw` | 每次 add 必写 | **不召回** | append-only；写入时按 `raw_trim` 裁剪工具输出 |
| L2 `FACT` | `l2_fact` | MemAgent 抽取 + reconcile | 主通道 | 原子事实，版本化、不可变 |
| L3 `SUMMARY` | `l3_summary` | 滚动摘要 / episode | 主通道 | 按 session / episode |
| L4 `IDENTITY` | `l4_identity` | 身份归纳 | 主通道（归入 normal） | README 标 reserved，但代码参与召回 |
| L5 `KNOWLEDGE` | `l5_knowledge` | — | 不召回 | reserved |
| L6 `SCHEMA` | `l6_schema` | System 2 / evolve | graph 专属通道 | 心智模型图层 |
| L7 `INTENTION` | `l7_intention` | System 1 即时抽取 | graph 专属通道 | 有 `valid_until`，过期降级为普通事实 |

> **文档与代码的差异**：README 把 L4 写为 "Reserved"，但 `reader_base.py:146-155` 的 `_VDB_RECALL_LAYERS` 明确包含 `L4_IDENTITY` 并参与语义+BM25 排序。**以实现为准**，README 该行为过时描述（记为 KNOWLEDGE_CONFLICT，见 06-validation）。

## 2. Chat 生命周期（写入）

```
add(data, scene=None)
  → _parse_input → content / messages / input_tokens
  → scene router（无 tool 短路 chat；含 tool 调 classify_messages_scene）
  → MemoryWriter.write
      ① layer_assign（默认 L1_RAW；可显式 extra.layer）
      ② embed → vector_store.upsert(L1)
         └─ 失败：仅记录 _l1_error，**non-fatal**，继续后续
      ③ 若 agent_mode=full 且 L1 成功：
           history_context（最近 k 轮）→ MemAgent.extract
           → reconcile(candidates)
           → 失败：返回 ReconcileFailure，**本批一条不落库**
           → 成功：按 ops 批量 embed → ADD/SUPERSEDE/UPDATE/DELETE
      ④ L3 summary / graph projection（异步、best-effort）
```

证据：`writer.py:2570-2760`；`writer.py:1298-1315`（reconcile fail-closed）；`writer.py:2670-2705`（L1 non-fatal）。

## 3. 版本化事实（L2）

### 3.1 状态机

```
（新事实）──ADD──▶ ACTIVE, is_latest=True
                     │
        新事实矛盾 ──SUPERSEDE──▶ 旧: SUPERSEDED + is_latest=False
                                 新: ACTIVE, supersedes=[old_id]
        合并吸收 ──ADD(supersedes=[]) + DELETE(old)──▶ 旧: SHADOW + is_latest=False
```

- DELETE 是**逻辑删除**：`status=SHADOW`、`is_latest=False`、保留在 VDB，不参与召回（`reconciler.py:18-26`；`models/memory.py:121-128`）。
- `SHADOW` 还用于"低置信度推断"。
- 候选去重命中时也置 SHADOW，且**连带历史链**（`reconciler.py:763-775`）。

### 3.2 链头与回溯

- reconcile 的候选只取链头（`is_latest=True`），`_trace_full_chain` 补全 `supersedes/superseded_by`，以链头为代表（`reconciler.py:634-658`）。
- 每轮只对当前有效版本做判断，避免对历史版本重复判重（`reconciler.py:987` 注释）。
- 原始出处：`source_raw_memory_id`（L1 → L2）、`evidence_chain`、`source_session_id/source_turn_index`（`models/memory.py:247-285`）。

## 4. Working / Coding 双层

### 4.1 表结构（`coding/store.py`）

```sql
-- working_fact（每 session 恰好 1 条）
fact_id, session_id, user_id, agent_id, workspace_id, branch,
statement, subject, files, details, source, created_at,
derived_experience_ids TEXT DEFAULT '[]'

-- coding_memory_meta（Experience，0..N）
experience_id, task, search_keys, solution, boundary_envs,
source_fact_ids TEXT DEFAULT '[]', confidence REAL DEFAULT 0.7, ...
```

### 4.2 双向链接

- `fact.derived_experience_ids` ↔ `experience.source_fact_ids`
- `_crosslink()` 读取双方现值后**取并集**再覆盖写（`coding/writer.py:290-325`）；单侧失败只 warning，不阻断另一侧。

### 4.3 阈值语义

- Fact：无阈值，始终写 1 条（"做过的事必须被记住"）。
- Experience：由抽取阶段的 value bar 决定；`coding/writer.py:188` 的日志文本 `no drafts produced (value bar)` 表明**阈值在 LLM 抽取判断中**，不是硬编码数值。
- 被取代的 fact 通过 `fact_chain_summary` 保留可追溯性（README 声明；本轮未定位到该字段的具体实现文件，列入 Candidate C-02）。

## 5. Coding 版本演化链（`coding/chain_store.py`）

### 5.1 数据模型

```sql
thread(id, owner, branch, file, symbol, pkey, lo, hi, n_rounds, updated_at, body)
round(id, thread_id, said, patch, at)
repo_symbol(...)
```

- `owner = "<user_id>|<workspace_id>"`，**必须进键**：否则不同用户/工作区的同名 branch/file/pkey 会串成一条 thread，造成数据泄漏（`chain_store.py:182-200`）。
- `pkey`：同一文件/符号上的多个独立决策点；不进键会把多个决策点串成一条链，版本序数失去语义。

### 5.2 写入触发

```
add(scene=coding) → writer.write() 成功
  → _observe_chain(messages, user_id, workspace_id, branch)   # best-effort
      → HyCodeMem.observe(text, meta)
          ├─ _pass_gate（噪音轮/判分轮拒收）
          ├─ _guess_branch（正则，去尾部标点）
          ├─ _get_patch：① 文本 diff → ② git show → ③ 编辑动作合成
          ├─ _split_hunks → (file, symbol, lo, hi)
          └─ _upsert_thread + _add_round
```

- 向量层写入失败（`success=False`）→ **跳过链**，避免把失败轮先记进演进链（`client.py:3030-3040`）。
- 链只依赖对话原文，与向量层结果解耦；链挂不上不影响 add 返回。

### 5.3 patch 三条来源与取舍

| 来源 | 场景 | 取舍 |
|---|---|---|
| 文本内 `diff --git` | 宿主把 diff 贴进对话 | 主路径，直接截取到 diff 结束标记 |
| `git show --unified=3 <commit>` | 事后灌库 / 轨迹导入 | 需要 `MEMORY_CODING_CHAIN_REPO_ROOT` 与 commit/key |
| 编辑动作合成 | commit 为空、git 反查落空 | 只认 `old='''…''' new='''…'''` 这类；sed/heredoc/口头确认**不猜**（"抽错比抽不到更坏"） |

合成 diff 的行号统一写 1（真实行号不可知）→ 按 `symbol` 归并，不按行区间（`chain_store.py:254-262`, `314-360`）。

### 5.4 失败版本防掺入

`_edit_failed()`：编辑动作后窗口内出现失败迹象，且其后没有成功迹象，才判失败并拒收；先失败后重试成功仍收（`chain_store.py:369-384`）。设计意图："链掺进一个没生效的版本会误导未来 agent，宁可漏收不要误收"。

### 5.5 读取形态

- `recall()` 返回链块，按 fact 挂载；链本身不参与排序（README "decorator, not a retriever"）。
- 最后一版完整代码存 `thread.body`，供 `_body_at` 取上下文。

## 6. 隔离维度总表

| 维度 | 作用域 | 强制点 |
|---|---|---|
| `user_id` | 全链路 | `writer.py` 校验必填；MemHub `owns()` 检查；user_id 禁止 `:` |
| `agent_id` | chat 记忆 | 默认 `default_agent`；owner=agent 时按 agent 判重 |
| `session_id` | episode / fact 归属 | 默认 `default_session` |
| `workspace_id` + `branch` | coding 记忆与链 | `owner` 键、store 查询条件 |
| `source` | 来源标注 | `normalize_source()`，与 agent_id 正交 |
