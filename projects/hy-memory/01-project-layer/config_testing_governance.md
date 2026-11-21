# 01.3 · Config, Testing & Governance

## 1. 配置面（`hy_memory/config.py` + `.env.example`）

### 1.1 运行模式
| 配置 | 默认 | 影响 |
|---|---|---|
| `HyMemoryClient(mode=...)` | `pro` | lite=零 LLM；pro=抽取；ultra=System 2 |
| `MEMORY_ENABLE_GRAPH` | `false` | pro 下是否启用图；ultra 强制开 |
| `MEMORY_ENABLE_INTENTION` | `true` | 是否抽取 L7 |
| `MEMORY_FEW_SHOT_ENABLED` | `false` | extractor/reconcile prompt 是否附 few-shot |
| `MEMORY_STRUCTURE_METADATA_ENABLED` | `false` | 结构化元数据 |
| `MEMORY_ENABLE_QUERY_REWRITE` | `false` | agentic search 的 query rewrite |

### 1.2 成本与可靠性
| 配置 | 默认 | 说明 |
|---|---|---|
| `MEMORY_LLM_MODEL` | `gpt-4.1-nano` | 任意 OpenAI 兼容模型 |
| `MEMORY_LLM_MAX_RETRIES` | 5 | LLM 重试 |
| `MEMORY_LLM_TIMEOUT` | 180 s | |
| `MEMORY_EMBEDDER_MAX_RETRIES` | 2 | |
| `MEMORY_EMBEDDER_TIMEOUT` | 15 s | 注释说明这是真实上限 |
| `MEMORY_EMBEDDER_BATCH_SIZE` | 64 | |
| `MEMORY_LLM_TEMPERATURE` | 0.1 | `none/omit` 可完全不发该参数（适配 kimi/deepseek 约束） |

### 1.3 存储
| 配置 | 默认 |
|---|---|
| `MEMORY_VECTOR_STORE` | `chroma` |
| `MEMORY_COLLECTION_NAME` | `agent_memories` |
| `MEMORY_EMBEDDING_DIMS` | 1536 |
| `MEMORY_DATA_DIR` | `~/.hy_memory` |
| 图 | Kuzu 默认 / Neo4j 可选 |
| Working/Coding 元数据 | 本地 SQLite |

### 1.4 去重
- `MEMORY_DEDUP_THRESHOLD` 默认 `0.95`（`writer.py:775-777`）。
- 意图批内去重：同日 `valid_until`（都 None 或日期相等）+ cosine ≥ 阈值 → 后者丢弃（`writer.py:756-825`）。

### 1.5 词面通道的"静默降级"
`pyproject.toml` 注释自述：
- `rank-bm25` 缺失 → lexical 通道**静默降级为纯稠密**（`lexical.py` 的 except 兜住，不报错），属"功能缺失而非故障"。
- `jieba` 缺失 → 仅含 CJK 时 lazy import，退化 bigram。
- 这两个依赖因此放进主依赖而非 optional。

## 2. 测试体系

| 项 | 值 |
|---|---|
| 测试文件 | 133 个 `test_*.py`，34,300 行 |
| 目录 | `tests/`（含 `tests/coding/` 23 个） |
| 框架 | pytest + pytest-asyncio（`asyncio_mode="auto"`） |
| 默认过滤 | `-m 'not integration'`，integration 需显式加入 |
| 标记 | slow / integration / asyncio |
| CI | `ci.yml`：PR/push 触发，`uv sync --extra dev --extra faiss --extra qdrant` → `pytest tests/ -q` |
| MemHub CI | Node 24 + pnpm，独立 job；有 `apps/core/tests` 与前端测试 |

**覆盖观察**：测试量与 `hy_memory`（10.3 万行）之比约 1:3；但本轮未统计各模块的测试密度，不能据此断言"覆盖充分"。`tests/coding/` 的存在说明版本链/生产力链路是重点验证对象。

## 3. 治理机制（MemHub）

### 3.1 管理面与引擎的权威边界
`memhub/docs/ARCHITECTURE.md` 明确三条原则：
1. 所有 memory 读写（含 backend 选择）交给 SDK。
2. 原子操作经 SDK；SDK 缺能力先补 SDK，禁止在 Core 直接改 VDB/Graph 或维护第二份正文。
3. MemHub 不存 memory 实体；只可持久化 `memory_id` 作为外键；禁止自判 supersede。

这不是口头约定——`api.py` 中所有读写走 `HyMemoryClient`；`source_grants` 等表只存管理元数据。

### 3.2 本地 API 防护（`auth.py`）
- Origin 必须属于 `{dev-origin, one-memory://app}` 或 loopback。
- loopback 判定使用 `ipaddress.ip_address(host).is_loopback`，**不 prefix 匹配**——防 `127.attacker.example` 经 DNS rebinding 解析到 127.0.0.1。
- Bearer token 与期望值用 `hmac.compare_digest`（常量时间比较）。
- 失败：`403 origin_not_allowed` / `401 invalid_local_token`。

### 3.3 来源授权（`imports/grants.py`）
- 表 `source_grants(user_id, source_id, requested_path, root_identity, revoked_at)`。
- `authorized_root()`：无行或 `revoked_at` 非空 → `None`；否则构造 `AuthorizedSourceRoot`，并校验 `root_identity_matches()`；不匹配抛 `SOURCE_IDENTITY_CHANGED`。
- 语义：**授权绑定的是"这个路径身份"，不是"这个字符串路径"**。目录被替换/重建后旧授权失效，需重新确认。
- 撤销是软删（写 `revoked_at`），保留审计痕迹。

### 3.4 多用户与作用域
- 默认用户 `local`；仅 `hermes` 允许接入第二个用户根（`MULTI_USER_SOURCES = {"hermes"}`）。
- 导入 lane：`lane_key(user_id, source_id)`；`local` 为裸 source_id（保持历史键不变），其他用户为 `source@user`。
- `user_id` 正则排除 `:`：SDK 用 `:` 拼 history key，允许冒号会读到别人的历史（`user_scope.py:20-25`）。
- MemHub 的 `owns(record_user_id, user_id)`：SDK 的 get/update/delete/history 不接收 user，故由 MemHub 在操作 id 前校验归属；历史数据 `local`/空都归 `local`。

### 3.5 workspace scope
- 格式 `workspace:v1:<sha256>`；设计要求"调用方必须显式传入 workspace scope"。
- 实测（文档内注明）：本机 536 条已确认记忆中 417 条只带 workspace scope，默认 recall 无法返回 → 新增 `workspace_resolution` 把调用方 `cwd` 翻译成各 source 会记录的身份。
- 隔离不被放宽：调用方只能命名自己所在目录，无法请求别人的 workspace。
- Cursor 的项目名会被截断并附自有短哈希 → 通过导入时已见名称的明文比对解析，不重建哈希。

### 3.6 安全报告
- `SECURITY.md`：要求走私密 GitHub Security Advisories，不在公开 issue 披露；**未承诺 SLA**。
- MemHub 另有 `memhub/SECURITY.md` 与 `PRIVACY.md`；仓库含 `SBOM.spdx.json`。

## 4. 配置/治理的关键风险

| 风险 | 证据 | 影响 |
|---|---|---|
| 词面通道静默降级 | `pyproject.toml` 注释 | 检索质量下降但无告警 |
| user_id 允许字符集依赖"排除 `:`"这一约定 | `user_scope.py` | 若 SDK 未来改用其他分隔符，隔离假设失效 |
| workspace scope 曾导致默认召回为空 | `workspace_resolution.py` 文档 | 历史数据需要迁移/兼容 |
| System 2 需手动 evolve | `client.py:2292` | 不调用则 L6 永不生成；无自动补偿 |
