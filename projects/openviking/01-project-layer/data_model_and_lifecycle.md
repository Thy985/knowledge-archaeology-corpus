# 01 · Project Layer — Data Model & Lifecycle

## 1. URI 命名空间

`viking://` 是统一逻辑根；映射到物理路径 `/local/{account_id}/...`（`05-storage.md`）。

| URI 前缀 | ContextType | 物理映射 | 语义 |
|---|---|---|---|
| `viking://resources/...` | `resource` | `/local/{account_id}/resources/...` | 账户内共享知识（文档/代码） |
| `viking://user/{user_id}/memories/...` | `memory` | `/local/{account_id}/user/{user_id}/memories/...` | 用户私有记忆 |
| `viking://user/{user_id}/skills/...` | `skill` | `.../user/{user_id}/skills/...` | 用户技能 |
| `viking://user/{user_id}/peers/{peer_id}/...` | memory/resource | 同上 | 访客/他者视图 |
| `viking://user/{user_id}/privacy/{category}/{target}/` | — | 同前缀 | 隐私值版本库 |
| `viking://~/memories`、`viking://~/skills` | 简写 | 当前用户根 | CLI 便捷别名 |

`ContextType` 枚举只有三个值：`skill | memory | resource`（`openviking/core/context.py:26-31`）。
`ContextLevel` 枚举：`ABSTRACT=0 | OVERVIEW=1 | DETAIL=2`（同文件 `:34-41`）。

## 2. 文件与记录模型

### 2.1 目录 sidecar

```
viking://resources/docs/auth/
├── .abstract.md      # L0：一句话摘要（≤ 50,000 bytes 的 memory abstract 上限见 05-storage.md）
├── .overview.md      # L1：结构与要点
└── *.md              # L2：原文
```

- sidecar 由 `SemanticProcessor` **异步、自底向上**生成（`parse/tree_builder.py:54` 注释）。
- sidecar 不是用户源文件：并发写用 `coalesce_version` + 写入前 `ExactPathLock`（`09-transaction.md`）。
- 检索不读 L0 文件本身：`abstract` 来自向量 payload（`context_assembler/tiers.py:5-7`）。

### 2.2 向量记录（context collection）

字段（`05-storage.md`）：`id / uri / parent_uri / context_type / is_leaf / vector / sparse_vector / abstract / name / description / created_at / active_count`；ACL 开启后追加 `acl_mode / acl_direct_grants / acl_inherited_grants`（`15-acl.md`，代码 `storage/acl.py:51`）。

- `abstract` 在 memory 类记录中存放**正文**（为了检索文本可读），因此走 50,000 字节上限；本地记录新格式改为 32-bit 长度打包 JSON，取消旧的 65,535 字节上限（`05-storage.md`）。
- `is_leaf` 区分目录摘要记录与叶子内容记录；`level` 由 URI 后缀映射：`.abstract.md→0`，`.overview.md→1`，其余→`2`（`observability/reporting/facts.py:266-275`）。

### 2.3 Memory 文件（`MemoryFile`）

`openviking/session/memory/dataclass.py:294`

```python
uri, content, links[], backlinks[], memory_type, extra_fields{}
```

- 序列化时 `version = old.version + 1`（`utils/memory_file_utils.py:32`）。
- `links/backlinks` 是**双向关系**：from 写 `links`，to 写 `backlinks`（`memory_updater.py:1797-1816`）。
- `extra_fields` 承载 schema 动态字段 + 系统字段（system-managed 字段在每次 Update 时保留，`memory_updater.py:1719-1728`）。

## 3. Memory 类型体系（schema 即产品）

内置模板目录：`openviking/system_assets/memory/memory_templates/builtin/`

| 模板 | 语义 | 目录 | 关键字段 / 合并 |
|---|---|---|---|
| `preferences` | 用户偏好 | `.../memories/preferences` | `user`/`topic`（immutable）+ `content`；按 topic 一文件 |
| `profile` | 用户画像 | `.../memories/profile` | 结构化字段 |
| `identity` | 身份 | `.../memories/identity` | — |
| `soul` | 价值观/人格 | `.../memories/soul` | — |
| `entities` | 实体 | `.../memories/entities` | 需外部参数（registry 注释 `:236`） |
| `events` | 事件 | `.../memories/events` | 唯一允许 deepen 到 `full` 的类别 |
| `experiences` | 可复用经验 | `.../memories/experiences` | `experience_name`（immutable）+ `content`（replace）+ `supersedes` |
| `cases` | 执行案例 | `.../memories/cases` | agent evolution |
| `trajectories` | 轨迹 | `.../memories/trajectories` | agent evolution |
| `skills` / `tools` | 技能/工具 | `.../skills`、`.../tools` | — |

代码常量：`CASE_MEMORY_TYPE="cases"`、`TRAJECTORY="trajectories"`、`EXPERIENCE="experiences"`、`EVENT="events"`；`AGENT_EVOLUTION_MEMORY_TYPES = {cases, trajectories, experiences}`（`session/memory/constants.py:9-13`）。

### 3.1 一个 YAML 模板的字段

`preferences.yaml`：

```yaml
memory_type: preferences
directory: "viking://user/{{ user_space }}/memories/preferences"
filename_template: "{{ user|lower }}/{{ topic|lower }}.md"
embedding_template: "{{ user }}\n\n{{ topic }}\n\n{{ content }}"
overview_template: "# Preferences Overview ..."
fields:
  - name: user      # merge_op: immutable
  - name: topic     # merge_op: immutable（兼身份键）
  - name: content   # merge_op: patch
```

`experiences.yaml` 额外有：`operation_mode: "upsert"`、`stage: agent`、`agent_only: true`。

### 3.2 merge_op 语义（`session/memory/merge_op/`）

| merge_op | 行为 | 代码 |
|---|---|---|
| `immutable` | 身份字段，禁止覆盖（用作文件名/身份键） | `immutable.py` |
| `replace` | 整段替换（如 experience content、supersedes） | `replace.py` |
| `patch` | 结构化补丁（`base.py:96` 明确 "All string fields with merge_op=patch use this structure"） | `patch.py` |
| `sum` | 数值累加 | `sum.py` |
| `link_merge` | 链接合并（去重、双向） | `link_merge.py` |

字段级 merge 失败时：**保留旧值 / 丢弃新值，继续处理其它字段**（`memory_updater.py:1716-1728`）。

## 4. 生命周期

### 4.1 资源导入（`add_resource`）

```
acquire final_uri 资源锁
  → 构建 R/N/F/V 快照（请求意图/解析产物/当前资源树/当前向量记录）
  → 编译 ContextUpdatePlan
  → 同步落地内容（content actions）
  → 清理解析临时产物
  → 入队 index / semantic 任务
  → 移交资源锁（或释放）
```

- 内容树**先于**派生层更新；成功后可能出现"内容已提交、摘要/向量尚未生成"的窗口。
- 自动命名：`exists(candidate)` 先检查；占用则 `_1/_2`；只对不存在候选尝试 `TreeLock`（非阻塞）。
- `rm` 试图对处理中资源加 `TreeLock` → `ResourceBusyError`。

### 4.2 会话提交（`session.commit`）

```
Phase 1（EXACT lock on session root）
  1 读 messages + policy，分配 archive number，切分 archived/retained
  2 持久化 archive messages + metadata
  3 入队 SessionCommitMsg + 创建 task
  4 持久化 retained live messages + session metadata + phase1 ready marker
  5 释放锁，返回 task_id（无可归档则 skipped）

Phase 2（后台，锁外）
  1 读 archived messages，生成 summaries
  2 抽取/更新 memories，调度派生处理
  3 写 memory_diff.json + completion marker
```

崩溃恢复：`SessionCommit` 持久队列（task type `session_commit`）在重启后续跑 Phase 2；**无独立 redo log**。
关键边界：模型调用可重试但**不保证生成文本一致**，所以 Phase 2 不是幂等重算，而是"基于已发布 archive 继续处理"。

### 4.3 记忆更新（`MemoryUpdater.apply_operations`）

```
precheck: operations.has_errors() → 整批返回
URI migration 物化 + 校验 → 失败整批返回
逐条 upsert（每条独立 try/except）
  └ _apply_upsert: 先读盘 → merge fields → 写盘
删除（多重守卫）
同步 resource refs
向量化（重读文件 → embedding）
链接写入（只对成功持久化的端点）
```

### 4.4 删除与移动

| 操作 | 顺序 | 锁 |
|---|---|---|
| `rm(file)` | 锁 → 删索引 → 删文件 → 解锁 | EXACT |
| `rm(dir)` | 锁 → 递归收集 URI → 删索引（recursive） → 删文件 | TREE |
| `mv(file)` | 双 EXACT → 复制 → 更新索引 → 删源 | EXACT ×2 |
| `mv(dir)` | 双 TREE → 复制 → 更新索引 → 删源 | TREE ×2 |

代码：`viking_fs/_ops.py:240-400`（rm）；文档 `09-transaction.md`（mv/add）。

### 4.5 队列与后台任务（部分清单）

`openviking/storage/queuefs/`：`add_resource`、`content_write`、`embedding`、`semantic`、`reindex`、`session_commit`、`external_task`。
队列语义：持久、可恢复、按 task type 路由；semantic 处理可 `adopt lock_handoff`。

## 5. 状态机（会话）

```
created → active → (commit) → archive_published(phase1)
                              → queued(session_commit)
                              → extracting(phase2)
                              → completed | failed
                              → (delete) deleted
```

未完成 → 完成判定不能只看目录存在；需查 task 终态 + archive completion marker（`09-transaction.md`）。

## 6. 数据一致性的三个"没有"

1. **没有跨存储原子事务**：VikingFS / VectorDB / QueueManager 之间没有 2PC。
2. **没有强一致 ACL**：ACL 随索引更新，同一 URI 的不同 level 记录可暂时持有不同版本。
3. **没有可重放的确定性抽取**：LLM 调用重试不保证同文，因此 Phase 2 只能续跑，不能"重算得到唯一结果"。

这三点是后续 KO-02 / KO-05 / KO-06 的事实底座。
