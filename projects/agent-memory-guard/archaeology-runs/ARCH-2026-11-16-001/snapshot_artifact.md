# Repository Snapshot — ARCH-2026-11-16-001

## 1. 快照事实

| 项 | 值 | 证据 |
|---|---|---|
| 项目 | OWASP Agent Memory Guard | `README.md` / `pyproject.toml` |
| 仓库 | `https://github.com/OWASP/www-project-agent-memory-guard.git` | `git remote -v` |
| 分支 | `main`（single-branch shallow clone） | `git branch --show-current` |
| Commit | `b63f76eece80d84387b6056919bd94816c60bd89` | `git log -1` |
| Commit 时间 | `2026-11-15 12:30:35 +0000` | `git log -1 --format=%ci` |
| Commit message | `chore: update download stats [skip ci]` | `git log -1 --format=%s` |
| 版本 | `0.4.0` | `pyproject.toml:7`；`src/agent_memory_guard/__init__.py:__version__` |
| License | Apache-2.0 | GitHub API license.spdx_id；`pyproject.toml` |
| 语言 | Python | 132 个 `.py` 文件 |
| 仓库体积 | 13 MB（工作树，含 `.git`） | `du -sh .` |
| 文件数 | 831（不含 `.git`） | `find` 计数 |
| Python 行数 | 21,160 行（132 文件） | `wc -l` |
| 测试文件 | 47 个 `test_*.py` | `find` 计数 |
| Python 要求 | `>=3.9`（classifier 3.9–3.14） | `pyproject.toml` |
| star 数 | 204（仅作背景，不参与选型） | GitHub API |
| 运行时间戳 | 本轮快照于 2026-11-16 生成 | 本文件 |

## 2. 基础地图

### 2.1 包与入口

| 入口 | 文件 | 说明 |
|---|---|---|
| Python 包 | `src/agent_memory_guard/` | 10,448 行（`src/` 全部 `.py`） |
| CLI `amg` | `src/agent_memory_guard/cli.py` → `cli:main` | `pyproject.toml [project.scripts]` |
| CLI `amg-bench` | `src/agent_memory_guard/bench/cli.py` → `bench.cli:main` | 同上 |
| HTTP 服务 | `src/agent_memory_guard/server.py`（431 行） | 可选 `server` extra：FastAPI/uvicorn/pydantic |
| MCP 服务 | `mcp-server/src/amg_mcp_server/server.py` | 独立 `mcp-server/pyproject.toml` |
| GitHub Action | `action.yml` | 仓库根 |
| Demo | `demo.py` | 仓库根 |

### 2.2 核心模块

| 模块 | 行数 | 职责 | 关键符号 |
|---|---:|---|---|
| `guard.py` | 1,653 | 统一读写/删除/晋升/快照/回滚门面；检测器编排；访问门；trace | `MemoryGuard`、`write`、`read`、`delete`、`promote`、`retire_if`、`snapshot`、`rollback`、`explain`、`_run`、`_enforce` |
| `identity.py` | 757 | 主体 id、`AgentHandle`、ambient identity（contextvars + 帧锚定 + GC 防护） | `ActingContext`、`AgentHandle`、`check_id`、`_AMBIENT`、`_Entry`、`ambient_for` |
| `policies/access.py` | 553 | 访问规则、选择器、决策顺序、registry/admin/class gate | `Selector`、`AccessRule`、`AccessPolicy`、`AccessDecision` |
| `policies/policy.py` | 513 | 内容策略（allow/redact/quarantine/block）、YAML 加载、protected/immutable keys | `Policy`、`PolicyRule`、`load_policy`、`with_access` |
| `classification.py` | 约 150 | 记忆来源类别与晋升图 | `MemoryClass`、`PromotionEdge`、`PromotionRules`、`ClassificationRegistry` |
| `events.py` | — | 安全事件与动作/严重度/来源枚举 | `SecurityEvent`、`Action`、`Severity`、`SourceClass` |
| `integrity.py` | — | SHA-256 基线 | `IntegrityRegistry`、`hash_value` |
| `storage/memory_store.py` | — | 默认进程内存储 | `InMemoryStore` |
| `storage/redis_store.py` | 231 | Redis 存储 | `RedisStore` |
| `storage/snapshots.py` | — | 环形快照缓冲（默认 50） | `Snapshot`、`SnapshotStore` |
| `middleware.py` | — | WSGI/ASGI 中间件接入 | — |
| `server.py` | 431 | HTTP API | — |
| `scanner/__init__.py` | 543 | 静态扫描（semgrep 规则 + 文件发现） | — |

### 2.3 检测器（默认套件）

`guard.py:__init__` 注册：

1. `PromptInjectionDetector`
2. `SensitiveDataDetector`
3. `SizeAnomalyDetector`
4. `RapidChangeDetector`
5. `ProtectedKeyDetector`（由 policy 的 protected/immutable keys 构造）
6. `CrossTaskContaminationDetector`
7. `SelfReinforcementDetector`

另有 `detectors/` 下可选/专用：`anomaly.py`、`cross_task.py`、`excessive_autonomy.py`、`injection.py`、`leakage.py`、`memory_persistence_injection.py`、`ml_injection.py`、`privilege_escalation.py`、`protected_keys.py`、`self_reinforcement.py`、`tool_abuse.py`。

### 2.4 核心数据结构与状态

| 状态 | 存储位置 | 生命周期 |
|---|---|---|
| 记忆值 | `MemoryStore`（默认 `InMemoryStore`，可选 Redis） | 进程/外部 |
| 类别标签 + 来源任务 | `ClassificationRegistry`（`_classes` / `_origin_task`） | 进程内；随 snapshot 捕获/回滚 |
| 完整性基线 | `IntegrityRegistry` | 进程内 |
| 最后写入者 | `MemoryGuard._written_by` | 进程内；随 snapshot 捕获/回滚 |
| 快照 | `SnapshotStore` 环形队列（默认 50） | 进程内 |
| 事件 | `MemoryGuard._events` 列表 | 进程内，随 guard 生命周期 |
| 隔离区 | `MemoryGuard._quarantine` dict | 进程内 |
| 当前任务 | `MemoryGuard._current_task` | 进程内 |
| ambient 身份 | `contextvars.ContextVar` + `_BY_ANCHOR` / `_BY_HANDLE` 注册表 | 进程内，按线程/任务/帧 |
| 操作 trace | `_OP` / `_LAST_TRACE`（contextvars） | 当前线程/任务 |

### 2.5 访问与治理机制（0.4.0 核心）

- **决策顺序**（`policies/access.py` 模块 docstring；`AccessPolicy.decide`）：
  1. registry：声明了 `principals` 时，未声明 id 直接拒绝；
  2. admin：`rollback` / `retire` / `snapshot` 必须命中 `admins`；
  3. rule：首个 key 命中的规则决定，写类操作看 `writers`，读看 `readers`；
  4. default：未被规则覆盖的 key 取 `default`（默认 `deny`）；
  5. class：写/删/晋升进入或离开受门控类别（默认 `POLICY`、`VERIFIED_PREFERENCE`）需要 `class_writers`。
- **检查时机**（`guard.py`）：`write` 在检测器之前；`read` 在 existence 检查之前；`delete` 在 store 触碰之前；`promote` 先 ACL、再晋升图、再 class；`rollback` 先 admin，再逐快照 class 差异检查。
- **身份来源**（`identity.py`）：显式 `principal=` / `AgentHandle` > ambient `with guard.as_agent(...)` > anonymous。文档明确“in-process identity 是调用方断言，不是认证”。
- **边界**：`AgentHandle` 不暴露 event log、quarantine、snapshots、classification；`snapshot()` / `rollback()` 只返回 id。
- **策略加载限制**（`policies/policy.py:_check_fields`）：YAML 中出现 `access`/`agents`/`principals` 等 per-agent 字段会 `ValueError`，访问控制只能通过 `Policy.with_access(...)` 编程构造。

### 2.6 测试体系

- `tests/` 47 个 `test_*.py`。
- 与本次重点直接相关：
  - `tests/test_access_control.py`（约 50 个用例：ACL、owner namespace、class gate、admin、rollback、深拷贝、pickle、并发）
  - `tests/test_identity_context.py`（约 35 个用例：ambient identity、asyncio/thread/generator/GC 边界）
  - `tests/test_redaction_fail_closed.py`（redaction 修复）
  - `tests/test_decision_trace.py`（explain/trace）
  - `tests/test_policy.py` / `test_policy_tiered.py` / `test_policy_yaml_guard.py`
  - `tests/test_guard.py` / `test_detectors.py` / `test_integrity.py` / `test_snapshots.py`
- `pyproject.toml` pytest `testpaths=["tests"]`；coverage 配置 omit 多个适配器/服务文件。

### 2.7 配置与外部依赖

- **必装**：`PyYAML>=6.0`。
- **可选 extras**：`server`（FastAPI/uvicorn/pydantic）、`ml`（transformers/torch）、`langchain`、`crewai`、`llamaindex`、`autogen`、`openai-agents`、`redis`、`test`、`dev`、`docs`。
- **策略配置**：YAML policy（`default_action`、`protected_keys`、`immutable_keys`、`rules`）；0.4.0 的 access 规则不在 YAML，需 `Policy.with_access`。
- **CI**：`.github/workflows/ci.yml`、`publish.yml`、`publish-mcp-server.yml`、`scorecard.yml`、`semgrep-self-scan.yml`、`update-downloads.yml`。

### 2.8 外部集成

`src/agent_memory_guard/integrations/`：`agno.py`、`autogen.py`、`crewai.py`、`langchain.py`、`llamaindex.py`、`openai_agents.py`；根目录 `integrations/` 另有框架适配示例/清单。

## 3. 0.3.3 → 0.4.0 变更对照（refresh 理由）

| 维度 | 归档 0.3.3（`8403091`） | 当前 0.4.0（`b63f76e`） |
|---|---|---|
| 版本 | 0.3.3 | 0.4.0 |
| 提交差 | — | `ahead_by=115`，300 changed files，`+420/−56` |
| 访问控制 | 无 per-agent ACL | `Policy.with_access(AccessRule(...))`，检测器前硬门 |
| 身份 | 无 principal | `guard.as_agent()` / `AgentHandle` / `principal=` / ambient contextvars |
| 命名空间 | 无 | `agents.{owner}.*` + `writers=["{owner}"]` |
| 角色/注册表 | 无 | `principals={id:[roles]}`；未声明 id 拒绝 |
| 管理面 | 无 | `admins` 控制 snapshot/rollback/retire；class gate 控制 POLICY/VERIFIED_PREFERENCE |
| 值语义 | 共享引用 | access 规则下写入/读取深拷贝；不可深拷贝对象 `TypeError` |
| 可解释性 | 无 | `explain()` dry-run；`trace=True` + `last_trace()` |
| 删除 | LlamaIndex 适配器可绕过 | `delete_message(s)` 抛 `AccessDenied`；guard entries 同步删除 |
| 脱敏 | 其他 detector 的 redacted finding 可能未脱敏存储/返回 | 按报告 detector 的 `redact()`；无 `redact()` 则 quarantine / 读回 `[REDACTED]` |

## 4. 快照结论

- 本快照仅基于 `main` @ `b63f76e` 的实际文件与 `git` 元数据；版本以 `pyproject.toml` 为准。
- 0.4.0 的核心新增不是“检测器数量”，而是**把访问控制从事后策略提升为操作前硬门**，并引入 principal/namespace/admin/class 四层治理；这正是 Memory 周 E 层（作用域）与 F 层（信任边界）需要的可执行证据面。
- 后续 Archaeology 需重点验证：ACL 是否在所有读写删晋升回滚路径真正前置；ambient identity 的线程/异步/生成器边界是否有绕过；class gate 是否可被 `write(cls=...)` 或 `promote` 绕过；redaction fail-closed 是否覆盖所有 detector。
