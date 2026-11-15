# 04 Flow Atlas — 七类流（49 Edge）

> 每条 Edge 都从真实代码导出，格式：`Edge ID | From → To | 触发/条件 | 证据（symbol/file:line）`。`condition` 是实际分支条件，不是文档描述。

## 4.1 Control Flow（8）

| ID | From → To | 触发/条件 | 证据 |
|---|---|---|---|
| CT-01 | `MemoryGuard.write()` → `_run()` | `principal is not None` 或 `_trace_on` 或 `_OP` 非空或 `_AMBIENT` 非空或 `_ANCHORED` 或 `policy.access is not None`（任一为真；否则走 0.3 快路径） | `guard.py:701-750` |
| CT-02 | `_run()` → `_resolve()` | 总是先解析身份 | `guard.py:337-355` |
| CT-03 | `_resolve()` → `gate(ctx)` | `ctx.principal is None && !trace && access is None && !op_state` 为假时进入慢路径 | `guard.py:337-360` |
| CT-04 | `gate()` → `_enforce(ctx, op, key)` | `access is not None`；否则 gate 不会被调用 | `guard.py:408-412` |
| CT-05 | `_enforce()` → `AccessPolicy.decide()` | 非 tracing 且 `access.allows()` 为真时直接返回；否则 `decide()` 走完整步骤 | `guard.py:408-437` |
| CT-06 | `decide()` allow → `body(ctx)` | 返回的 `AccessDecision.allowed=True`；否则 `_deny()` 抛 AccessDenied | `guard.py:360-380`, `policies/access.py:445-544` |
| CT-07 | `body(ctx)` → `_write_impl()` | write 的 body 先 `_detach`（有 access）再调用 `_write_impl` | `guard.py:752-777` |
| CT-08 | `promote()` → graph → class gate → commit | `edge` 存在；`requires_verification` 满足；class gate 允许 | `guard.py:556-671` |

## 4.2 State Flow（7）

| ID | From → To | 触发/条件 | 证据 |
|---|---|---|---|
| ST-01 | write → store committed | `decision` 为 ALLOW 或 REDACT（非 BLOCK/QUARANTINE） | `guard.py:900-930` |
| ST-02 | write → classification set | `target_class is not None`；`task_id` 取新值或保留原 origin | `guard.py:930-950` |
| ST-03 | delete → store/integrity/classification/self-reinforcement 清理 | 非 protected key；access/class gate 通过 | `guard.py:1103-1125` |
| ST-04 | snapshot → 捕获 store/class/origin/writer | `_capture(label)` 在 `snapshot()` 或 pre-block/pre-retire 调用 | `guard.py:513-555` |
| ST-05 | rollback → 恢复 store/class/writer | 快照存在；admin gate；`_rollback_class_check` 通过 | `guard.py:1216-1332` |
| ST-06 | quarantine → `_quarantine[key]` | decision=QUARANTINE 或 redaction unredactable | `guard.py:820-840, 860-900` |
| ST-07 | retire → 逐 key 删除 | predicate 返回真；非 protected；caller 可改该 class | `guard.py:1155-1195` |

## 4.3 Data Flow（8）

| ID | From → To | 触发/条件 | 证据 |
|---|---|---|---|
| DF-01 | caller value → `_detach(value)` | 有 access policy；深拷贝失败抛 TypeError | `guard.py:752-760` |
| DF-02 | store value → `_detach(value)` | 有 access policy；reader 得到独立副本 | `guard.py:983-1004` |
| DF-03 | value → `canonical_serialize` → SHA-256 | `hash_value()` 调用；dict 按 key 排序、`default=str` | `integrity.py:9-30` |
| DF-04 | value → `_stringify` → redact → str | `SensitiveDataDetector.redact()` 对容器先展平 | `detectors/leakage.py`；本轮探针 |
| DF-05 | snapshot data → deepcopy | `SnapshotStore.capture()` | `storage/snapshots.py:42-60` |
| DF-06 | event metadata → `SecurityEvent.metadata` | `_emit(metadata=...)`；可含 source/trace/verdicts | `events.py`；`guard.py:_emit` |
| DF-07 | backing store messages → `get_messages` return | `GuardedChatStore.get_messages()` 直接 `list(raw)` | `integrations/llamaindex.py:88-93` |
| DF-08 | HTTP JSON `value` → `_guard.write` | `/write` 请求体 `WriteRequest.value`；无 principal 传递 | `server.py:114-170, 264-300` |

## 4.4 Evidence Flow（7）

| ID | From → To | 触发/条件 | 证据 |
|---|---|---|---|
| EV-01 | access deny → `SecurityEvent(access_control, HIGH, BLOCK)` | `_deny(decision)` | `guard.py:439-455` |
| EV-02 | detector verdict → event | BLOCK/QUARANTINE/REDACT/READ_ALLOW_WITH_FINDINGS | `guard.py:800-920, 1017-1090` |
| EV-03 | integrity mismatch → `SecurityEvent(integrity, CRITICAL, BLOCK)` + raise | `_read_impl` 捕获 IntegrityError | `guard.py:1005-1020` |
| EV-04 | quarantine → event | `decision=QUARANTINE` 或 unredactable | `guard.py:820-840, 860-900` |
| EV-05 | redaction → event | `decision=REDACT` 成功或 withheld | `guard.py:900-930, 1017-1058` |
| EV-06 | snapshot → event/metadata | `_capture` 后写入 `metadata.pre_snapshot_id` 或 event metadata | `guard.py:513-555, 1185-1195` |
| EV-07 | decision trace → `_LAST_TRACE` / event metadata | `trace=True` 时 `_step` 累积，`finally` 写入 `_LAST_TRACE` | `guard.py:320-390` |

## 4.5 Authority Flow（7）

| ID | From → To | 触发/条件 | 证据 |
|---|---|---|---|
| AU-01 | 显式 `principal=` → `ActingContext(via="explicit")` | `explicit is not None` 且不是 `_HandleId` | `guard.py:309-318` |
| AU-02 | `with guard.as_agent()` → ambient context | `_AMBIENT` 有该 guard 的 entry；`ambient_identity_possible` 为真 | `identity.py:568-757`；`guard.py:309-318` |
| AU-03 | `AgentHandle` → `_HandleId` → `via="handle"` | handle 方法传 `principal=self._pid` | `identity.py:600-648` |
| AU-04 | registry → unknown principal denied | `principal not in reg` | `policies/access.py:466-470` |
| AU-05 | admin ops → admins only | `operation in ADMIN_OPS`；无 admins 则 deny | `policies/access.py:470-480` |
| AU-06 | gated class → `class_writers` | `writes=True` 且 `current/target class` 有 `_class_sel` | `policies/access.py:489-518` |
| AU-07 | adapter `.guard` / `_guard` → raw authority | 属性/私有字段可访问；无 gate | `integrations/llamaindex.py:63-64`；`identity.py:568` |

## 4.6 Memory Flow（6）

| ID | From → To | 触发/条件 | 证据 |
|---|---|---|---|
| MF-01 | guard write → `store.set(key, value)` | ALLOW/REDACT 且非 BLOCK/QUARANTINE | `guard.py:900-910` |
| MF-02 | guard read → `store.get(key, default)` | key 存在；integrity 通过 | `guard.py:983-1004` |
| MF-03 | immutable key → `integrity.baseline()` | 首次写且 `policy.is_immutable(key)` | `guard.py` 构造 + `_write_impl`；`integrity.py:33-50` |
| MF-04 | snapshot → ring buffer | `capture()`；`len > max_snapshots` 丢最旧 | `storage/snapshots.py:42-70` |
| MF-05 | event → `_events` list | 每次 `_emit()`；无上限 | `guard.py:_emit` |
| MF-06 | quarantine → `_quarantine` dict | QUARANTINE/unredactable；无上限 | `guard.py:820-840, 860-900` |

## 4.7 Policy Flow（6）

| ID | From → To | 触发/条件 | 证据 |
|---|---|---|---|
| PF-01 | YAML `Policy.from_dict()` → `_check_fields()` | 加载时；per-agent 字段 → ValueError | `policies/policy.py:455-499` |
| PF-02 | `Policy.strict()` → protected/immutable keys | 构造 policy；写 protected key → detector；delete → `_protected_detector.matches` | `policies/policy.py:180-200`；`guard.py:1103-1115` |
| PF-03 | promotion graph → promote decision | `edge is None` 或 `requires_verification` 不满足 → ClassificationError | `classification.py:42-63`；`guard.py:610-650` |
| PF-04 | `AccessRule` `{owner}` → 动态 owner | key 匹配出 `{owner}` segment；owner 必须合法 id | `policies/access.py:200-280`；`tests/test_access_control.py:229-256` |
| PF-05 | no matching rule → `default` | `_decide_rule` 末尾；默认 deny | `policies/access.py:530-544` |
| PF-06 | `AMG_POLICY` env → server policy | 启动时 `_policy_map`；未知值 fallback strict | `server.py:93-101` |

## Flow → KO 交叉校验
- CT-01/03/04/06 + EV-01 + AU-04/05/06 支撑 **KO-01**（gate-before-effect）；CT-04 的 `_deny` 先于 `body` 是硬证据。
- DF-04 + EV-05 + ST-06 支撑 **KO-02**（unhandled → withhold/quarantine）。
- EV-02 + MF-05 支撑 **KO-03**（fail-open 但可见）。
- PF-03 + AU-06 + ST-05 支撑 **KO-04**（图 + 门禁）。
- DF-07 + AU-07 + MF-01 支撑 **KO-05**（最弱等价路径）。
- PF-01 + AU-04 支撑 **KO-06**（配置 fail closed）。
- AU-01/02/03/07 支撑 **KO-07**（能力面分离）。
- MF-04/05/06 + EV-03/04 支撑 **KO-08**（可观测性即权限面）。
- PF-01 与 CT-08 的文档/实现差异支撑 **KO-09**。
- PF-06 与 EV-02 的版本 pin 支撑 **KO-10**。
- **矛盾边**：DF-07（Llama 直读）与 CT-04（gate 在 body 前）不能同时成立为“所有读都受控”；已在 KO-05 与 C-01 中显式记录，不强行 PASS。
