# 01 Project Layer — OWASP Agent Memory Guard 0.4.0

## 1.1 项目定位与边界
AMG 是**库优先、服务为辅**的进程内记忆防护层：
- 主入口是 `MemoryGuard`，调用方在写/读/删/晋升时显式经过它。
- 附带一个 FastAPI 服务（`amg serve`）用于本地扫描/演示，但**不含认证**，也不是多租户服务。
- 提供 LlamaIndex / LangChain / CrewAI / Agno 四个适配器，把 guard 包在框架的 chat store / memory 接口外面。
- 提供 `amg-bench` 记忆系统基准（AMSB），以及一个 MCP server，把扫描器暴露为工具。

**它不做什么**：不实现 memory 后端（用 dict-like store 接口）；不做跨进程认证；不做网络层鉴权；不替换 agent framework 的路由。

## 1.2 仓库结构（实际目录）
```
src/agent_memory_guard/
├── guard.py                 # 核心：MemoryGuard、write/read/delete/promote/snapshot/rollback/retire
├── identity.py              # AgentHandle、ambient identity（contextvars + frame/opcode）
├── classification.py        # MemoryClass、PromotionGraph、ClassificationRegistry
├── integrity.py             # canonical JSON + SHA-256 baseline
├── events.py                # SecurityEvent、Action、Severity、Event handlers
├── middleware.py            # ASGI middleware（scan/write/read 旁路）
├── server.py                # FastAPI app + request/response models
├── cli.py                   # amg / amg-bench 入口
├── detectors/
│   ├── injection.py         # PromptInjectionDetector
│   ├── leakage.py           # SensitiveDataDetector（含 redact）
│   ├── anomaly.py           # SizeAnomalyDetector、RapidChangeDetector
│   ├── protected_keys.py         # ProtectedKeyDetector
│   ├── contamination.py     # CrossTaskContaminationDetector
│   ├── self_reinforcement.py # SelfReinforcementDetector
│   ├── privilege_escalation.py / tool_abuse.py / excessive_autonomy.py / memory_persistence_injection.py
├── policies/
│   ├── access.py            # AccessRule / Selector / AccessPolicy / AccessDecision
│   ├── policy.py            # Policy、YAML loader、protected/immutable keys
│   └── ...
├── storage/
│   ├── memory_store.py      # InMemoryStore
│   └── snapshots.py         # Snapshot / SnapshotStore（bounded 50）
├── integrations/
│   ├── llamaindex.py        # GuardedChatStore
│   ├── langchain.py         # GuardedChatMessageHistory
│   ├── crewai.py            # GuardedMemory
│   └── agno.py              # GuardedMemory
└── mcp-server/              # 独立包/工具封装
tests/                       # 47 个 test_*.py
docs/architecture/           # design.md / threat-model.md / ...
benchmarks/                  # security_benchmark.py + results + memory-systems
```

## 1.3 入口与生命周期
| 入口 | 位置 | 行为 |
|---|---|---|
| `MemoryGuard(policy=..., store=..., detectors=..., trace=False)` | `guard.py:120-240` | 构造默认 7 detector、integrity、classification、snapshot store、event handlers |
| `guard.write(key, value, ...)` | `guard.py:701` | 解析身份 → gate → 深拷贝（有 access 时）→ detectors → 决策 → commit/quarantine/raise |
| `guard.read(key, default, ...)` | `guard.py:950` | 解析身份 → gate → 存在性 → integrity → 取值/深拷贝 → detectors → 决策/redact/withhold |
| `guard.delete(key)` | `guard.py:1072` | gate（writer + class）→ protected key 检查 → 删 store/integrity/class/self-reinforcement |
| `guard.promote(key, target, verified=...)` | `guard.py:556` | key-rule gate → promotion graph → class gate → 更新 class |
| `guard.snapshot/rollback` | `guard.py:1196/1216` | admin gate；rollback 额外做 class 变更检查 |
| `guard.retire_if(predicate, reason)` | `guard.py:1121` | admin gate → 先 `_capture(pre-retire)` → 逐 key 跳过 protected/无权 class → predicate |
| `guard.explain(op, key, ...)` | `guard.py:243` | dry-run，返回 `AccessDecision`，不执行 |
| `agent_memory_guard.server:app` | `server.py:83` | FastAPI；`/scan`、`/write`、`/read`、`/events`、`/stats`、`/reset` |
| `amg` / `amg-bench` | `pyproject.toml:47-50` | console scripts |

**典型生命周期**：
1. 应用启动时构造 `Policy`（strict 或自定义 access），构造 `MemoryGuard`，把 `AgentHandle`（不是 raw guard）交给每个 agent/tool。
2. agent 在 `with guard.as_agent("id")` 内或通过 handle 调 write/read。
3. write 先过 access gate（拒绝则不跑 detector、不写 store），再跑 detector，按最严重 action 决定 allow/redact/quarantine/block。
4. read 先过 access gate（拒绝不暴露 key 是否存在），再 integrity verify，再跑 outbound detector，必要时 redact/withhold。
5. 晋升通过 promotion graph；gated class 还需 `class_writers`。
6. 运维侧通过 events/quarantine/verify_all/list_snapshots 观察；这些接口不经过 access gate，属于 raw guard 权限。

## 1.4 核心数据结构
| 结构 | 字段/语义 | 位置 |
|---|---|---|
| `MemoryClass` | `ephemeral` / `user_preference_candidate` / `verified_preference` / `retrieved_fact` / `tool_observation` / `policy` | `classification.py:14-27` |
| `PromotionEdge` | source/target/requires_verification；默认图 3 条边 | `classification.py:42-63` |
| `ClassificationRegistry` | `_classes` + `_origin_task`，可 export/import（snapshot 恢复） | `classification.py:80-140` |
| `AccessRule` | name、keys、writers、readers；支持 `*`、role、`{owner}`、`<anonymous>` | `policies/access.py:60-240` |
| `AccessPolicy` | rules、default、admins、class_writers、principals、ambient_identity | `policies/access.py:289-544` |
| `AccessDecision` | allowed/operation/key/principal/via/rule/pattern/owner/reason/stage/steps | `policies/access.py:150-190` |
| `SecurityEvent` | timestamp、detector、severity、action、operation、key、message、metadata、principal、source_class、receipt_uri | `events.py` |
| `Snapshot` | snapshot_id/timestamp/label/data/digest/metadata；`data` 是 store 全量深拷贝 | `storage/snapshots.py:14-30` |
| `SnapshotStore` | `OrderedDict`，默认 max_snapshots=50，超出丢最旧 | `storage/snapshots.py:32-90` |
| `Action` / `Severity` | allow/redact/quarantine/block；LOW/MEDIUM/HIGH/CRITICAL | `events.py` |
| `SourceClass` | USER_INPUT / EXTERNAL_TOOL / AGENT_AUTHORED / SYSTEM / UNKNOWN | `detectors/self_reinforcement.py` 等 |

## 1.5 状态与存储
- **memory store**：默认 `InMemoryStore`（dict-like `set/get/delete/keys/items`），可注入自定义 store；guard 只依赖该协议。
- **integrity registry**：key → SHA-256 baseline；baseline 在 key 首次写且 `policy.is_immutable(key)` 时建立；构造函数也会为已有 immutable key 建 baseline。后续 write 不更新 baseline。
- **classification registry**：key → MemoryClass + origin task；snapshot 会一起捕获/恢复。
- **snapshot store**：进程内 `OrderedDict`，默认 50；`digest` 仅取证记录，rollback 不校验。
- **quarantine**：`dict[key, value]`，进程内，无容量上限；不进 store。
- **events**：`list[SecurityEvent]`，进程内，无容量上限；`MemoryGuard(events=[...])` 可注入。
- **written_by**：key → last writer principal；snapshot 捕获/恢复；delete/retire 清理。

## 1.6 测试体系
- 47 个 `test_*.py`，504 passed / 2 skipped / 10 xfailed（默认、allow-all、trace 三种模式均同）。
- 覆盖重点：access control（顺序、selector、owner、registry、class gate、rollback）、identity context（线程/task/generator/GC）、redaction fail-closed、decision trace、classification/promotion、protected keys、integrity、snapshots、retire、middleware、API scanner、LlamaIndex/LangChain/CrewAI/Agno 适配器（可选依赖缺失时 skip）。
- 10 个 xfail 是项目自报未实现的攻击族，集中在 `tests/benchmarks/test_delayed_activation.py` 等。

## 1.7 配置与策略
- `Policy.strict()`：protected keys（`identity.*`、`system.*`、`agent.goal`）不可写/删，immutable keys 建 baseline。
- `Policy.permissive()`：默认放行，仍记录事件。
- `Policy.tiered()`：按风险分层。
- YAML：`rules`、`protected_keys`、`immutable_keys` 等；**per-agent 字段（access/principals/agents/writers/readers/principal 等）在 0.4.0 会 `ValueError`**，只能通过 `Policy.with_access()` 编程式配置。
- 环境变量：`AMG_POLICY=strict|permissive|tiered`、`AMG_TEST_MODE=allow-all|trace`、`AMG_CORS_ORIGINS`（默认关闭；`*` 直接 `SystemExit`）。

## 1.8 权限与治理机制（0.4.0 新增）
```
principal 解析
  ├─ 显式 principal= / AgentHandle._HandleId  → 直接使用（断言，不认证）
  ├─ ambient（with guard.as_agent）            → 仅当 AccessPolicy.ambient_identity=True
  └─ 否则 anonymous
        ↓
AccessPolicy.decide
  ├─ registry：声明了 principals 时，未声明 id 直接拒绝
  ├─ admin：rollback/retire/snapshot 必须是 admins；无 admins 则无人可做
  ├─ rule：第一条 key 匹配的 AccessRule；write/delete/promote 看 writers，read 看 readers
  ├─ default：无规则覆盖时 default（默认 deny）
  └─ class：write/delete/promote 触及 gated class 时还需 class_writers
```
- `admins` 默认空；`admins=["<anonymous>"]` 才恢复 0.3 的匿名管理行为。
- `class_writers={"verified_preference": ["*"]}` 可单独开一个 class，不会顺带开 POLICY。
- `Selector` 的 `role:` 只能来自 registry，调用方不能自带角色。
- `check_id` 限制 1–64 字符 `[A-Za-z0-9_][A-Za-z0-9_-]{0,63}`，禁止点号/glob，避免 id 替换进 key pattern 时扩大作用域。
- `AccessRule` 的 `{owner}` 必须是完整 key segment，`{owner}` 前缀必须全 literal；一个 pattern 最多一个 `{owner}`。

## 1.9 外部依赖与运行环境
- Python >=3.9；`pyproject.toml` 版本 `0.4.0`；License Apache-2.0。
- 唯一运行依赖 `PyYAML>=6.0`；测试依赖 pytest、httpx 等。
- 可选：`llama-index-core`、`langchain-core`、`crewai`、`agno`；缺失时相关测试 skip。
- 无网络调用；无外部服务；无 API key。
- 仓库 `repo size` 4,707 KB、831 files、132 Python files、21,160 Python LOC、47 个 `test_*.py`。
- `git` 浅克隆 HEAD `b63f76e`；CI matrix 声明 3.9–3.14（实际本轮只跑 3.12）。

## 1.10 版本与发布
- `pyproject.toml` `version = "0.4.0"`。
- tag `v0.4.0` → `b8353d605582`；旧 tag `v0.3.3` → `a8a2d37d0149`。
- `CHANGELOG.md` 0.4.0 日期 2026-10-07；0.3.3 及更早受 redaction 安全修复影响。
- GitHub 仓库 stars 204、created 2026-02-16、pushed 2026-11-15、default branch `main`、未 archived。
