# 04 · Flow Atlas（七类流）

> 约定：每条 Edge 必须有 `symbol` / `file` / `condition` / `failure_path` 四个可核验字段。
> 本 Atlas 覆盖**写入与读取主干**，不覆盖 System 2 内部算法与 MemHub 前端。

## 1. Control Flow（控制流）

| Edge | From → To | symbol / file | condition | failure path |
|---|---|---|---|---|
| CF-01 | API 入口 → scene 归一化 | `_normalize_scene` · `client.py:861` | 任意 add 调用 | 非法 scene → `ValueError` |
| CF-02 | scene 归一化 → 路由判定 | `async_add` · `client.py:2930` | scene ∈ {None, chat, working, coding} | — |
| CF-03 | 路由判定 → chat（短路） | `async_add` · `client.py:2949` | messages 无 tool **或** scene=chat | 无（零 LLM） |
| CF-04 | 路由判定 → judge | `classify_messages_scene` · `coding/judge.py:154` | scene=None 且含 tool | LLM 异常 → chat |
| CF-05 | judge → coding writer | `_get_coding_writer` · `client.py:3010` | judge 返回 coding/working | — |
| CF-06 | judge → chat | `async_add` · `client.py:3004` | judge 返回 chat / 异常 / 解析失败 | warning |
| CF-07 | 强制 coding → chat | `async_add` · `client.py:2958` | scene=coding 但无 tool | warning + 回退 |
| CF-07b | 强制 coding → chat（**静默**） | `async_add` · `client.py:2976` | `coding_available=False`（lite / 非 messages / 无 coding 配置 / `coding.enable=false`） | **无日志**，直接走 chat |
| CF-08 | writer → reconcile | `_reconcile_and_store_locked` · `writer.py:1259` | agent_mode=full 且有候选 | `ReconcileFailure` → 整批不落库 |
| CF-09 | writer → coding 链 | `_observe_chain` · `client.py:3038` | variant=coding 且 `resp.success` | 静默跳过 |
| CF-10 | evolve → System 2 | `evolve` · `client.py:2292` | mode=ultra | lite/pro → RuntimeError |

## 2. State Flow（状态流）

| Edge | From → To | symbol / file | condition | failure path |
|---|---|---|---|---|
| SF-01 | 新事实 → ACTIVE | `reconcile.apply` · `reconciler.py:397` | action=ADD | 写库失败 → 整批回滚（逻辑上未落库） |
| SF-02 | ACTIVE 旧 → SUPERSEDED | 同上 | action=SUPERSEDE | 旧节点 `is_latest=False` |
| SF-03 | ACTIVE 旧 → SHADOW | 同上 | action=DELETE / 去重命中 | 不物理删 |
| SF-04 | SHADOW/SUPERSEDED → 召回不可见 | `_VDB_RECALL_LAYERS` + 状态过滤 · `reader_base.py:1555` | 任意 search | — |
| SF-05 | L7 intention 过期 → 普通事实 | `intention` 归档逻辑（README 声明） | `valid_until` 过期 | 文档声明，代码路径本轮未逐行验证 |
| SF-06 | coding fact → superseded 链 | `fact_chain_summary`（README 声明） | 同 session 新 fact 取代旧 fact | 未定位实现（C-02） |

## 3. Data Flow（数据流）

| Edge | From → To | symbol / file | condition | failure path |
|---|---|---|---|---|
| DF-01 | 输入 → L1 raw | `writer.write` · `writer.py:2570` | 任意非 lite 写入 | non-fatal |
| DF-02 | L1 raw → embed | `embed_service` | `vector_store` 可用 | 失败 → `_l1_error` |
| DF-03 | 对话 → MemAgent 抽取 | `_run_agent` · `writer.py:3265` | agent_mode=full | 失败 → 无候选，跳过 reconcile？ |
| DF-04 | 候选 → L2/L3 节点 | `reconcile.apply` | ops 非空 | `ReconcileFailure` |
| DF-05 | 对话 → WorkingFact | `coding/writer.py` | variant ∈ working/coding | 写库失败 → 返回 error |
| DF-06 | 对话 → Experience | `coding/writer.py:212` | 抽取产生 drafts | 无 draft → 仅 fact |
| DF-07 | 对话 → patch | `_get_patch` · `chain_store.py:314` | 有 diff / commit / old-new | 三来源皆空 → 跳过本轮 |
| DF-08 | patch → thread/round | `_upsert_thread` · `chain_store.py:464` | `_pass_gate` 通过 | 拒收 |

## 4. Evidence Flow（证据流）

| Edge | From → To | symbol / file | condition | failure path |
|---|---|---|---|---|
| EF-01 | L1 raw → L2 fact | `source_raw_memory_id` · `models/memory.py:247` | 抽取产生 fact | 指针缺失 → 无法回溯原文（未强制非空） |
| EF-02 | 抽取结果 → evidence_chain | `MemoryNode.evidence_chain` | 节点构造时 | 空链表仍可写入 |
| EF-03 | fact → chain（代码版本） | `HyCodeMem.recall` 挂载 | fact 含 file/symbol | 无匹配 thread → 无块 |
| EF-04 | 用户原话 → round.said | `_add_round` · `chain_store.py:546` | pass gate | 原文未裁剪（可能含密钥，见 C-05） |

## 5. Authority Flow（权威流）

| Edge | From → To | symbol / file | condition | failure path |
|---|---|---|---|---|
| AF-01 | Host 请求 → 来源授权 | `authorized_root` · `grants.py:143` | 需要读某个 source | 无 grant → `None`； revoked → `None` |
| AF-02 | 授权 → 身份校验 | `root_identity_matches` · `grants.py:143` | grant 非空 | 不匹配 → `SOURCE_IDENTITY_CHANGED` |
| AF-03 | 导入 → lane 键 | `lane_key` · `user_scope.py:35` | 任意导入 | user_id 非法 → 拒绝 |
| AF-04 | 读取 id → 归属校验 | `owns` · `user_scope.py:57` | get/update/delete/history | 不归属 → 拒绝 |
| AF-05 | 浏览器 → 本地 API | `local_api_guard` · `auth.py:47` | 任意 HTTP 请求 | Origin 非白名单 → 403；token 错 → 401 |
| AF-06 | cwd → workspace scope | `workspace_resolution` | 调用方声明目录 | 无法解析 → 拒绝（不得猜测） |
| AF-07 | 管理面 → SDK | MemHub `api.py` | 任意记忆操作 | SDK 不可用 → 管理面报错，不降级自写 |

## 6. Memory Flow（记忆流）

| Edge | From → To | symbol / file | condition | failure path |
|---|---|---|---|---|
| MF-01 | 写入 → 会话 episode | `episode_summary` · `agent/episode_summary.py` | episode 启用 | 摘要失败 → 跳过 |
| MF-02 | episode → L3 summary | 同上 | 达到轮次阈值 | 同上 |
| MF-03 | 全量 VDB → L6 schema | `system2_agent` · `pipelines/system2_agent.py` | evolve() | 失败 → 返回 error，不自动重试 |
| MF-04 | coding fact ↔ experience | `_crosslink` · `coding/writer.py:290` | 有 fact 与 experience | 单侧失败 → warning |
| MF-05 | working/coding 存储隔离 | `_variant_db_path` · `coding/store.py:388` | variant 切换 | 后缀错误 → 读到另一业务库（静默） |

## 7. Policy Flow（策略流）

| Edge | From → To | symbol / file | condition | failure path |
|---|---|---|---|---|
| PF-01 | 写入 → 去重策略 | `MEMORY_DEDUP_THRESHOLD` · `writer.py:775` | 意图批内 | 阈值缺失 → 默认 0.95 |
| PF-02 | 抽取 → 价值门槛 | value bar prompt（`coding/writer.py:188`） | experience 生成 | 未过门槛 → 0 条 |
| PF-03 | 写入 → 失败策略 | `ReconcileFailure` · `writer.py:76` | reconcile 失败 | 整批拒绝 |
| PF-04 | 检索 → 层白名单 | `_VDB_RECALL_LAYERS` · `reader_base.py:146` | 任意 search | 不在白名单 → 永不召回 |
| PF-05 | 模式 → 能力门 | `mode != "ultra"` → `RuntimeError` · `client.py:2310` | evolve | 拒绝 |
| PF-06 | 链 → 失败编辑过滤 | `_edit_failed` · `chain_store.py:369` | 编辑动作 | 判失败 → 不入链 |

## 8. Flow → KO 交叉校验

| KO | 依赖的 Flow Edge | 校验结果 |
|---|---|---|
| KO-01 | CF-01~CF-07 | ✅ 路由五分支均可在代码定位 |
| KO-02 | CF-08, DF-01, DF-04, PF-03 | ✅ 但"DF-03 抽取失败是否进入 reconcile"未验证（C-03） |
| KO-03 | SF-01~SF-04, DF-04, PF-04 | ✅ |
| KO-04 | DF-05, DF-06, MF-04 | ✅ |
| KO-05 | DF-07, DF-08, EF-03, EF-04, PF-06, CF-09 | ✅ |
| KO-06 | AF-01, AF-02, AF-05, AF-07 | ✅ |
| KO-07 | AF-03, AF-04, AF-06, MF-05, SF-06 | ⚠️ SF-06 未定位实现，KO-07 的 chat 侧隔离证据略弱 |

**统计**：CF 11 + SF 6 + DF 8 + EF 4 + AF 7 + MF 5 + PF 6 = **47 条 Edge**。

> 本轮 Independent Validation 发现 `coding.enable=False` 的静默降级路径（CF-07b），Edge 数由 46 修正为 47。初稿自评 "24" 为估算，已废止。
