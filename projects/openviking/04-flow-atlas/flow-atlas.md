# 04 · Flow Atlas（七类流）

> 每条 Edge 必须可回溯：`symbol / file / condition / state transition`。
> `source` 标注：`code`（代码可证）/ `doc`（仅文档声明）/ `rust-unread`（依赖未读 Rust 实现）。
> id 规则：`FC`=Control，`FS`=State，`FD`=Data，`FE`=Evidence，`FA`=Authority，`FM`=Memory，`FP`=Policy。

---

## 1. Control Flow（22 edges）

| ID | From → To | Symbol / Condition | File:Line | Source |
|---|---|---|---|---|
| FC-01 | HTTP request → auth plugin | `resolve_identity()`；按 `auth_mode` 选 plugin | `server/auth/__init__.py:170-188` | code |
| FC-02 | auth plugin → RequestContext | `_build_request_context()`；评估 `deletion in progress` | `server/auth/__init__.py:150-166` | code |
| FC-03 | route → FSService | `ls/mkdir/rm/mv/tree/stat/read/abstract/overview/grep/glob` | `service/fs_service.py:129` | code |
| FC-04 | SearchService → IntentAnalyzer | `if query_planner is None: raise ValueError` | `retrieve/intent_analyzer.py:38+` | code |
| FC-05 | IntentAnalyzer → HierarchicalRetriever | 产出 `{reasoning, queries}`；解析失败抛 `ValueError` | 同上 | code |
| FC-06 | Retriever → vector search | `search_type=="keywords"` → `search_by_keywords_in_tenant`；否则 `search_in_tenant` | `retrieve/hierarchical_retriever.py:102-240` | code |
| FC-07 | Retriever → rerank | `if reranker and not image_query`；候选 ×2 | 同上 + `params.py:41` | code |
| FC-08 | Retriever → context assembler | 按 URI 取最高分去重后进入 tier 解析 | 同上 | code |
| FC-09 | tier=abstract → vector payload | 零磁盘读 | `context_assembler/tiers.py:5-7` | code |
| FC-10 | tier=overview → `.overview.md` | `_overview_uri()` | `tiers.py:131` | code |
| FC-11 | tier=full → 原文 | `read_file`；并发 8 | `params.py:38` | code |
| FC-12 | SessionService.commit → Phase 1 | EXACT lock on session root | `09-transaction.md` | doc |
| FC-13 | Phase 1 → SessionCommit 队列 | 入队 `SessionCommitMsg` + task_id | `queuefs/session_commit_processor.py:25` | code+doc |
| FC-14 | 队列 → Phase 2 processor | 消费 task，锁外执行 | 同上 | code |
| FC-15 | Phase 2 → SessionCompressorV3 | `extract_long_term_memories` | `session/compressor_v3.py:425` | code |
| FC-16 | Compressor → ExtractLoop | `_extract_user_memories` → orchestrator.run | `compressor_v3.py:668` | code |
| FC-17 | ExtractLoop → MemoryUpdater | `submit(MemoryUpdateRequest)` | `session/memory/streaming_memory_updater.py:105` | code |
| FC-18 | ResourceService → ContextUpdatePlan | R/N/F/V 快照 → 编译 plan | `09-transaction.md` | doc |
| FC-19 | Plan → content write（同步） | final_uri 资源锁内 | 同上 | doc |
| FC-20 | content write → enqueue index/semantic | 内容落地后入队 | 同上 | doc |
| FC-21 | rm → `vector_store.delete_uris` | stat → lock → ACL MANAGE → 估数 | `storage/viking_fs/_ops.py:240+` | code |
| FC-22 | delete_uris → `_async_agfs.rm` | 索引删成功后 | 同上（顺序） | code |
| FC-23 | `_rm` → `viking_fs.rm(no_vector=True)` | `NamespaceProcessingMode.FILE_ONLY`（仅 `viking://system`） | `service/fs_service.py:651-657` | code |

## 2. State Flow（12 edges）

| ID | From → To | Trigger / Guard | File | Source |
|---|---|---|---|---|
| FS-01 | anonymous → resolved identity | api_key 或 trusted header + root key | `auth/plugins/*.py` | code |
| FS-02 | session created → active | 写入消息 | `session_service.py` | code |
| FS-03 | active → archive_published | Phase 1 marker 写完 | `09-transaction.md` | doc |
| FS-04 | archive_published → queued | `SessionCommitMsg` 入队 | `session_commit_processor.py` | code+doc |
| FS-05 | queued → extracting | worker 取任务 | 同上 | code |
| FS-06 | extracting → completed / failed | 产出 diff 或异常（默认吞异常为 completed+空） | `compressor_v3.py:425` | code |
| FS-07 | memory version n → n+1 | 每次成功写入 | `utils/memory_file_utils.py:32` | code |
| FS-08 | acl_mode none → inherit | manage 者切换 | `15-acl.md` | doc |
| FS-09 | inherit → restricted | manage 者切换 | 同上 | doc |
| FS-10 | restricted → inherit | manage 者切换（不能回 none） | 同上 | doc |
| FS-11 | acl disabled → enabled | 配置热/重启加载 | `_access.py:473` | code |
| FS-12 | encryption disabled → enabled | 配置加载 | `crypto/config.py:32-43` | code |

## 3. Data Flow（10 edges）

| ID | From → To | 载体 / 变换 | File | Source |
|---|---|---|---|---|
| FD-01 | 原始文件/URL → AGFS 内容树 | parser → tree builder | `parse/tree_builder.py` | code |
| FD-02 | 内容 → 向量记录 `abstract` | 生成/复用摘要（memory 类为全文） | `05-storage.md` + `params.py:67-76` | doc+code |
| FD-03 | `MemoryFile` → 磁盘文件 | `to_metadata()` 序列化 + version | `dataclass.py:294-344` | code |
| FD-04 | 磁盘文件 → 向量索引 | `_vectorize_memories` 重读后 embedding | `memory_updater.py:1296` | code |
| FD-05 | archived messages → summary → memory diff | Phase 2 抽取 | `compressor_v3.py:425` | code |
| FD-06 | privacy values → placeholder → 正文 | `PrivacySanitizer` | `privacy/sanitizer.py:28` | code |
| FD-07 | placeholder → 真值 | `PrivacyRestorer` | `privacy/restore.py:18` | code |
| FD-08 | R/N/F/V 快照 → ContextUpdatePlan | plan 编译 | `09-transaction.md` | doc |
| FD-09 | 旧 URI → 新 URI | `_materialize_uri_migrations` + link remap | `memory_updater.py:992+` | code |
| FD-10 | memory content → embedding text | YAML `embedding_template` | `preferences.yaml` | code |

## 4. Evidence Flow（6 edges）

| ID | From → To | 内容 | File | Source |
|---|---|---|---|---|
| FE-01 | 每次写入 → version 字段 | 单调递增，供审计/回滚判断 | `memory_file_utils.py:32` | code |
| FE-02 | Phase 2 → `memory_diff.json` | 记录新增/更新/删除 | `09-transaction.md` | doc |
| FE-03 | Phase 2 → completion marker | 完成后写，供查询 | 同上 | doc |
| FE-04 | 抽取异常 → warning 日志 | `"V3 memory extraction failed..."` | `compressor_v3.py:425` | code |
| FE-05 | ACL 变更 → 索引字段 | `acl_direct_grants/inherited/mode` | `storage/acl.py:51` | code |
| FE-06 | 请求 → metrics/observability | lifecycle/archive metric | `session_service.py:198-213` | code |

## 5. Authority Flow（8 edges）

| ID | From → To | 条件 | File | Source |
|---|---|---|---|---|
| FA-01 | root key → 可声明任意 account/user | `hmac.compare_digest(api_key, root_api_key)` | `trusted.py:139-151` | code |
| FA-02 | header account/user → 有效身份 | 与 URL 显式值必须一致，否则 InvalidArgument | `trusted.py:153-172` | code |
| FA-03 | admin path / key-only `/health` → ROOT | `_is_trusted_admin_path` / `is_control_probe` | `trusted.py:183-200` | code |
| FA-04 | direct + inherited → effective | `mode=="restricted"` 时丢弃 inherited | `15-acl.md` | doc |
| FA-05 | 父权限变更 → inherited 刷新 | 持续刷新，非一次性拷贝 | 同上 | doc |
| FA-06 | manage 者 → copy 不携带源 ACL | 复制后重新授权 | 同上 | doc |
| FA-07 | 目录 delete/move → 只校验目录 manage | 不逐子孙 | 同上 | doc |
| FA-08 | 加密开关 → provider 加密 | 默认 local/关闭 | `crypto/config.py:32-43` | code |
| FA-09 | MANAGE 预检 → `bypass_acl=True` | 递归删除内部，避免二次检查 | `service/fs_service.py:641-642` | code |

## 6. Memory Flow（9 edges）

| ID | From → To | 条件 / 变换 | File | Source |
|---|---|---|---|---|
| FM-01 | messages → operations | `ExtractLoop` ReAct + repair | `session/memory/extract_loop.py:117` | code |
| FM-02 | operations → resolved ops | 解析 URI/类型/字段；失败可 `resolution_skip` | `memory_updater.py:992+` | code |
| FM-03 | resolved → upsert | 逐条独立 try/except | `_apply_upsert:1614` | code |
| FM-04 | upsert → 重读磁盘 → merge | 字段级 `merge_op` | 同上 | code |
| FM-05 | merge 结果 → 写文件 + version++ | 系统字段保留 | 同上 | code |
| FM-06 | delete → 六守卫 | 失败则跳过 | `memory_updater.py`（`failed_link_migration_sources`） | code |
| FM-07 | 删除后 → 向量化 | 跳过 sidecar / 已删 URI | `memory_updater.py:1296` | code |
| FM-08 | 写入成功 → links/backlinks | 只对成功端点写关系 | 同上 | code |
| FM-09 | 新经验 → supersedes 旧经验 | 最终 merge 决定删除 | `experiences.yaml:50-57` | code |

## 7. Policy Flow（7 edges）

| ID | From → To | 策略 | File | Source |
|---|---|---|---|---|
| FP-01 | `agent_evolution_enabled` → 类型允许集 | false 时剔除 cases/trajectories | `compressor_v3.py:425`；`constants.py:9-13` | code |
| FP-02 | `strict_extract_errors` → 失败语义 | false：warning + 空；true：抛出 | 同上 | code |
| FP-03 | `acl.enabled` → 访问检查开关 | false：字段照写、检查短路 | `_access.py:473` | code |
| FP-04 | `DEPTH_CEILING_BY_CATEGORY` → 检索深度 | 仅 events 可到 full | `params.py:78-82` | code |
| FP-05 | `PURPOSE_PRESETS` → 候选桶配额 | coding 等用途分配槽位 | `params.py:86+` | code |
| FP-06 | `FULL_BODY_ABSTRACT_CATEGORIES` → 层级替换规则 | memory 类 overview 不得替代 abstract | `params.py:67-76` | code |
| FP-07 | `encryption.enabled` → 加密策略 | 默认关闭 | `crypto/config.py:35` | code |

---

## 8. Flow → KO 交叉校验

| KO | 依赖的关键 Edge | 校验结果 |
|---|---|---|
| KO-01 | FC-21, FC-22, FD-04, FD-01, FC-20 | 通过：写出/删除方向一致 |
| KO-02 | FM-06, FM-03, FM-05, FC-18, FC-19 | **部分**：`add_resource` 覆盖路径未验证（C-01） |
| KO-03 | FC-09, FC-10, FC-11, FP-04, FP-05, FP-06 | 通过 |
| KO-04 | FD-03, FD-10, FM-02, FM-04, FM-09 | 通过 |
| KO-05 | FC-12, FC-13, FC-14, FS-03~06, FE-02, FE-03 | **部分**：幂等性未验证（C-02） |
| KO-06 | FA-01~FA-04, FP-03, FS-08~11 | 通过 |
| KO-07 | FP-02, FC-04, FC-05, FM-02 | **通过但含张力**（EK-23 vs EK-24） |
| KO-08 | FC-03, FC-06, FD-01, FP-04 | 通过 |

## 9. Flow 统计

| 流 | Edge 数 | code | doc | 备注 |
|---|---:|---:|---:|---|
| Control | 23 | 17 | 6 | — |
| State | 12 | 7 | 5 | — |
| Data | 10 | 8 | 2 | — |
| Evidence | 6 | 5 | 1 | — |
| Authority | 9 | 8 | 1 | — |
| Memory | 9 | 9 | 0 | — |
| Policy | 7 | 7 | 0 | — |
| **合计** | **76** | **61** | **15** | — |

> 76 个 Edge 中 61 条由代码直接支撑；15 条仅文档声明（集中在事务/ACL/锁），全部标 `doc`，未混入 Fact。
> **未覆盖**：Rust `ragfs` 内部锁实现、embedding 队列消费者、gateway 路由 —— 见 Validation 的 Coverage 声明。
