# 02.1 · Evidence Index（32 条）

> 约定：`Fact` = 代码/文档中可直接读到的事实；`Observation` = 由多条代码路径组合得出的、尚未运行验证的判断。
> 所有行号基于 `main@9737431c`。

| ID | 类型 | 内容 | 证据 |
|---|---|---|---|
| F-01 | Fact | `_normalize_scene` 只接受 chat/working/coding，非法抛 ValueError | `hy_memory/client.py:861-890` |
| F-02 | Fact | `_scene_variant(None)` 返回 `working`，注释自述为兼容 fallback | `client.py:892-911` |
| F-03 | Fact | messages 无 tool 时 O(1) 短路走 chat，不调用 LLM | `client.py:2949-2952`（注释与分支） |
| F-04 | Fact | 含 tool 且 scene=None 时调用 `classify_messages_scene` 做三分类 | `client.py:2998-3010` |
| F-05 | Fact | judge 异常 → `logger.warning` 且 `is_coding=False`（走 chat） | `client.py:3001-3005` |
| F-06 | Fact | judge 内部 LLM 失败 / 解析失败 → 返回 "chat" | `hy_memory/coding/judge.py:215-230` |
| F-07 | Fact | 强制 `scene="coding"` 但无 tool → 回退 chat 并 warning | `client.py:2955-2965` |
| F-08 | Fact | 强制 `scene="chat"` 时 strip 所有 tool 消息 | `client.py:2944-2948` |
| F-09 | Fact | working 与 coding 共用 pipeline/schema，但独立 SQLite 与向量集合 | `client.py:3010`；`README.md:226-234` |
| F-10 | Fact | L1 raw 先 upsert；持久化失败只记录 `_l1_error`，继续执行 | `pipelines/writer.py:2660-2705` |
| F-11 | Fact | `user_id` 缺失时返回 400，不落库 | `writer.py:2605-2612` |
| F-12 | Fact | `agent_mode = "disabled" if mode == "lite" else "full"` | `client.py:3081` |
| F-13 | Fact | reconcile 失败时不执行 ADD-all 兜底，整批 0 条落库，由调用方重跑 | `writer.py:1298-1315` |
| F-14 | Fact | DELETE 执行 `status=SHADOW, is_latest=False`，不物理删除 | `agent/reconciler.py:18-26`；`models/memory.py:121-128` |
| F-15 | Fact | 候选只取链头（is_latest），`_trace_full_chain` 补全 supersedes/superseded_by | `reconciler.py:634-658` |
| F-16 | Fact | 判重只对 L2_FACT + L4_IDENTITY，按 owner 分组（user 跨 agent，agent 隔离） | `reconciler.py:763-800` |
| F-17 | Fact | 去重阈值 `MEMORY_DEDUP_THRESHOLD` 默认 0.95 | `writer.py:775-777` |
| F-18 | Fact | 主 VDB 召回层 = L2/L3/L4；L0 走独立隐藏通道（上限 10） | `pipelines/reader_base.py:136-155` |
| F-19 | Fact | 图重排失败只 warning，non-fatal | `reader_base.py:653` |
| F-20 | Fact | `evolve()` 仅 ultra 可用，非幂等、无队列、全量扫描；lite/pro 抛 RuntimeError | `client.py:2292-2332` |
| F-21 | Fact | `working_fact` 每 session 1 条，含 `derived_experience_ids`；Experience 表含 `source_fact_ids` | `coding/store.py:73-120` |
| F-22 | Fact | `_crosslink` 读取双方现值后取并集覆盖写；单侧失败仅 warning | `coding/writer.py:290-325` |
| F-23 | Fact | 无 experience draft 时日志 `no drafts produced (value bar)` | `coding/writer.py:188` |
| F-24 | Fact | 版本链 `observe()` 零 LLM、零 embedding，仅 sqlite3 | `coding/chain_store.py:227-240` |
| F-25 | Fact | `owner = "<user_id>|<workspace_id>"`，注释说明不进键会跨用户串链 | `chain_store.py:182-200` |
| F-26 | Fact | patch 三来源：文本 diff → git show → 编辑动作合成；合成 diff 行号写 1，按 symbol 归并 | `chain_store.py:314-360` |
| F-27 | Fact | 失败编辑（失败迹象且其后无成功迹象）不入链 | `chain_store.py:369-384` |
| F-28 | Fact | `_git()` 失败/异常返回空串，不抛错 | `chain_store.py:386-394` |
| F-29 | Fact | 向量层 `success=False` 时跳过 `_observe_chain`；链挂不上不影响 add | `client.py:3030-3045` |
| F-30 | Fact | MemHub 不存 memory 正文/不判 supersede，只存 `memory_id` 外键 | `memhub/docs/ARCHITECTURE.md` 原则 1-3 |
| F-31 | Fact | 来源授权读取时校验 `root_identity`，不匹配抛 `SOURCE_IDENTITY_CHANGED` | `memhub/apps/core/src/one_memory/imports/grants.py:143-170` |
| F-32 | Fact | 本地 API 用 `ipaddress.is_loopback` 判 loopback，Bearer 用 `hmac.compare_digest` | `memhub/apps/core/src/one_memory/auth.py:33-75` |
| F-33 | Fact | `user_id` 正则排除 `:`；lane_key 用 `@`；仅 hermes 支持多用户 | `memhub/apps/core/src/one_memory/user_scope.py:20-50` |
| F-34 | Fact | workspace scope 为 `workspace:v1:<sha256>`，可由调用方 cwd 解析，且只能解析自己所在目录 | `memhub/apps/core/src/one_memory/workspace_resolution.py:1-45` |
| F-35 | Fact | CI 在 PR/push 触发，pytest 默认排除 integration；MemHub 独立 job | `.github/workflows/ci.yml` |
| F-36 | Fact | `rank-bm25` 缺失时 lexical 通道静默降级为纯稠密 | `pyproject.toml:20-27` |

> 计数修正：本表实际 36 条（初稿自评 32 条的估计偏低）。Overview 的 Facts 以本表为准，后续自评已同步为 36。
