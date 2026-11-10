# 02 · Evidence Index（36 条事实）

> 证据强度：S0 Opinion / S1 Discussion / S2 Design Proposal / S3 Implemented / S4 Test Validated / S5 Runtime Validated。
> **本轮未运行代码/测试**，因此最高为 S3；文档独有结论标 **S2-doc**；厂商自报标 **S0-vendor**。
> `Rust-unread` 表示结论依赖 `crates/ragfs` 内部实现但本轮未读。

| ID | 事实（可证伪陈述） | 证据位置 | 强度 | 备注 |
|---|---|---|---|---|
| EV-01 | 上下文分三类：`skill / memory / resource` | `openviking/core/context.py:26-31` | S3 | 枚举定义 |
| EV-02 | 层级为 `ABSTRACT=0 / OVERVIEW=1 / DETAIL=2` | `core/context.py:34-41` | S3 | 与 L0/L1/L2 同义 |
| EV-03 | 目录摘要 sidecar 为 `.abstract.md` / `.overview.md` | `parse/tree_builder.py:54`；`observability/reporting/facts.py:266` | S3 | 异步生成（注释） |
| EV-04 | 检索默认 tier 为 `abstract`，仅 `events` 可升到 `full` | `retrieve/context_assembler/params.py:56-82` | S3 | `DEPTH_CEILING_BY_CATEGORY={"events":"full"}` |
| EV-05 | memory 类的 `abstract` 存全文，overview 反而更浅 | `params.py:67-76`（`FULL_BODY_ABSTRACT_CATEGORIES`） | S3 | 层级语义反转 |
| EV-06 | rerank 候选放大倍数为 2 | `params.py:41`（`RERANK_CANDIDATE_MULTIPLIER=2`） | S3 | 与检索器共用常量 |
| EV-07 | tier 读并发为 8 | `params.py:38` | S3 | `READ_CONCURRENCY=8` |
| EV-08 | `HierarchicalRetriever.retrieve` 为单次向量召回 + 可选 rerank，不是递归树遍历 | `retrieve/hierarchical_retriever.py:102-240` | S3 | 命名与机制反差 |
| EV-09 | rerank 仅对非 `image_query` 生效 | 同上（`if reranker and not image_query` 分支） | S3 | — |
| EV-10 | 图像查询未指定 level 时默认 `level=[2]` | 同上 | S3 | — |
| EV-11 | collection 不存在时检索返回空，不抛错 | 同上（`collection_exists_bound()` 早返回） | S3 | fail-soft |
| EV-12 | `IntentAnalyzer` 缺少 `query_planner` 时抛 `ValueError` | `retrieve/intent_analyzer.py:38` 起 | S3 | 硬依赖 LLM |
| EV-13 | 意图解析失败抛 `Failed to parse intent analysis response` | 同上 | S3 | 无降级 |
| EV-14 | 默认检索目录按 context_type 决定 | `core/retrieval_targets.py:51-90` | S3 | memory→user_root；resource→`viking://resources`+user_root |
| EV-15 | peer 视图会附加 `peers/{actor_peer_id}/...` 目录 | 同上 `:60-80` | S3 | 隔离 + 他者记忆 |
| EV-16 | `session.commit` Phase1 持 session root EXACT 锁，Phase2 后台 | `docs/en/concepts/09-transaction.md` | **S2-doc** | 代码未逐行验证锁获取点 |
| EV-17 | Phase2 经 `SessionCommit` 持久队列续跑，无独立 redo log | `storage/queuefs/session_commit_processor.py:25` + doc | S2/S3 | 队列类存在 |
| EV-18 | 归档切分 archived/retained，retained 为 active 原文 | `09-transaction.md` | S2-doc | — |
| EV-19 | 抽取异常在 `strict_extract_errors=False` 时只记 warning 返回空 | `session/compressor_v3.py:425-537`（外层 except） | S3 | **静默降级** |
| EV-20 | `agent_evolution_enabled=False` 时剔除 `cases/trajectories` | `compressor_v3.py:425`；`memory/constants.py:9-13` | S3 | — |
| EV-21 | memory 类型由 YAML 模板定义目录/文件名/embedding/字段 | `system_assets/memory/memory_templates/builtin/*.yaml` | S3 | 13 个模板 |
| EV-22 | `preferences` 以 `user + topic` 为身份键，一 topic 一文件 | `preferences.yaml`（`filename_template`） | S3 | immutable 字段 |
| EV-23 | `experiences` 强制三段式 `Situation / Approach / Reflect` | `experiences.yaml`（content 字段描述） | S3 | 机器指令式 prompt |
| EV-24 | `experiences` 有 `supersedes` 字段用于经验替代 | `experiences.yaml:50-57` | S3 | 版本链 |
| EV-25 | 字段级合并操作有 immutable/replace/patch/sum/link_merge | `session/memory/merge_op/` 目录 | S3 | — |
| EV-26 | merge 单字段失败保留旧值并继续 | `memory_updater.py:1716-1728` | S3 | best-effort |
| EV-27 | `_apply_upsert` 每次先从磁盘重读最新内容再合并 | `memory_updater.py:1614` 起（注释 "consecutive patches ... see each other's changes"） | S3 | 防 lost update |
| EV-28 | URI 迁移目标已存在 → `ConflictError("Memory rename target already exists")` | `memory_updater.py` `_apply_upsert` 内 | S3 | — |
| EV-29 | `apply_operations` 先物化+校验 URI 迁移，失败整批返回 | `memory_updater.py:992` 起 | S3 | — |
| EV-30 | 同批次 delete + upsert 同名 → 视为 Update，跳过 delete | `memory_updater.py`（日志 `"Replace-with-same-name treated as Update"`） | S3 | — |
| EV-31 | unresolved upsert 带 `resolution_skip` 时报告型跳过 | `memory_updater.py`（reason_code：INVALID_PEER_ID / INVALID_RANGES / PAGE_ID_TYPE_MISMATCH） | S3 | — |
| EV-31b | **批级**守卫：`has_unresolved_upserts` 为真时整个 delete 循环跳过，日志 `"Skipped delete because batch contains unresolved upsert URIs"`（独立审计补正） | `memory_updater.py` "Apply delete operations" 段 | S3 | 补充 |
| EV-32 | 删除在 upsert 之后执行；链接迁移失败会回滚并跳过对应 delete | `memory_updater.py`（`failed_link_migration_sources`） | S3 | 多守卫 |
| EV-33 | 向量化在删除之后执行，跳过 `.overview.md`/`.abstract.md` 与同批已删 URI | `memory_updater.py:1296` 调用点 | S3 | 派生层顺序 |
| EV-34 | `MemoryFile` 字段仅 uri/content/links/backlinks/memory_type/extra_fields | `session/memory/dataclass.py:294-344` | S3 | — |
| EV-35 | 版本号写回为 `old.version + 1` | `session/memory/utils/memory_file_utils.py:32` | S3 | — |
| EV-36 | `rm` 先 `vector_store.delete_uris` 再 `_async_agfs.rm` | `storage/viking_fs/_ops.py:240` 起（rm 实现段） | S3 | 顺序可证 |
| EV-37 | `rm` 对目录用 tree 锁、文件用 exact 锁；锁冲突→`ResourceBusyError("Resource is being processed")` | `_ops.py` rm 段 + `_access.py` | S3 | — |
| EV-38 | `rm` 目标不存在时走 `_finish_missing_delete` 清孤儿索引（幂等） | `_ops.py` | S3 | docstring 写 idempotent |
| EV-39 | `mv` 顺序：复制 → 更新索引 → 删源（文档） | `09-transaction.md` | S2-doc | 代码未逐行核 |
| EV-40 | `add_resource` 在 final_uri 建资源锁，先落内容再入队 index/semantic | `09-transaction.md` | S2-doc | — |
| EV-41 | 语义处理期间同路径 `rm` 冲突 | `09-transaction.md` | S2-doc | — |
| EV-42 | 锁文件 `.exact.ovlock.{hash}`，token `{owner_id}:{time_ns}:{lock_type}` | `09-transaction.md` | S2-doc | **Rust-unread** |
| EV-43 | 锁协议新旧版本不得混跑，需停写→排空→换代 | `09-transaction.md` | S2-doc | 迁移纪律 |
| EV-44 | 路径必须绝对，拒绝空/相对/重复斜杠/`.`/`..`/NUL | `09-transaction.md` | S2-doc | — |
| EV-45 | `Tree(/docs/a.md)` 不扩展为 `Tree(/docs)`；Exact 不保护目录子孙 | `09-transaction.md` | S2-doc | 锁粒度语义 |
| EV-46 | ACL 仅作用于 `viking://resources/...` | `15-acl.md`；`storage/acl.py` | S2-doc + S3 | is_acl_uri 存在 |
| EV-47 | principal 仅 `user:{id}`/`group:{id}`/`user:*`，不支持 `group:*` | `15-acl.md` | S2-doc | — |
| EV-48 | 有效权限 = direct + (mode=="restricted" ? ∅ : inherited) | `15-acl.md` | S2-doc | — |
| EV-49 | `acl_mode` ∈ none/inherit/restricted；不能回到 none 绕过父权限 | `15-acl.md` | S2-doc | — |
| EV-50 | `acl.enabled` 默认 false，关闭时仍写字段但不检查 | `15-acl.md` + `_access.py:473` | S2/S3 | — |
| EV-51 | ACL 随索引更新，同 URI 多 level 可短暂多版本，读不强一致 | `15-acl.md` | S2-doc | 显式一致性声明 |
| EV-52 | copy 不携带源 ACL；目录 delete/move 仅校验目录 manage | `15-acl.md` | S2-doc | — |
| EV-53 | trusted 模式必须 `X-API-Key == server.root_api_key`（`hmac.compare_digest`） | `server/auth/plugins/trusted.py:139-151` | S3 | 安全关键 |
| EV-54 | trusted 模式 header `X-OpenViking-Account/User` 与 URL 显式值不一致 → InvalidArgument | `trusted.py:153-172` | S3 | — |
| EV-55 | `X-OpenViking-Role` 仅允许 `user/admin`，且需 root key | `trusted.py:74-90` | S3 | — |
| EV-56 | 管理路径与 `/health` key-only 探针返回 ROOT 身份 | `trusted.py:183-200` | S3 | — |
| EV-57 | 多租户 account 外层、user 内层；ROOT/ADMIN/USER 三级 | `11-multi-tenant.md` | S2-doc | — |
| EV-58 | 隐私值按 `category + target_key` 分区，`current.json` + `history/version_N.json` | `13-privacy.md` | S2-doc | — |
| EV-59 | 隐私 upsert 是全量快照；与当前相同则不建版本 | `13-privacy.md` | S2-doc | — |
| EV-60 | 隐私写时占位、读时恢复 | `13-privacy.md` + `privacy/sanitizer.py` / `restore.py` | S2/S3 | — |
| EV-61 | 加密默认关闭，provider 默认 local | `crypto/config.py:32-43` | S3 | — |
| EV-62 | 三个 console script：`ov` / `openviking-server` / `openviking-gateway` | `pyproject.toml:234-237` | S3 | — |
| EV-63 | Python 733,129 行 / Rust 130,408 行 / 553 文件（py+rs+md 去重口径见 Snapshot） | `wc -l` | S3 | — |
| EV-64 | 803 个测试文件；37 个 CI workflow | `find` / `ls` | S3 | — |
| EV-65 | 厂商自报 LoCoMo 80–83%、token −34.3~91.0%、tau2 +6.87/+11.87pp | `README.md` | **S0-vendor** | **禁入 Fact/KO** |

## 统计

- 共 66 条证据登记（超出目标 36，保留原始粒度以便回溯；含独立审计补正 EV-31b）。
- 强度分布：S3 实现类 39；S2-doc 类 26；S0-vendor 1。
- 无 S4/S5（未运行测试/服务）。

> 与 `02-engineering-knowledge` 的 EK 不是一一对应：EK 是对多条 EV 的解释性归纳，允许 1 EK ← n EV。
