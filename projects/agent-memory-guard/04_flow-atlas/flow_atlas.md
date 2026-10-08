# 04 Flow Atlas — 七类流（从真实代码导出，关键 Edge 可回溯）

## 1. Control Flow（控制流）
```
agent 操作
  ├─ write(key, value, source_class, cls, task_id)                guard.py:250
  │    ├─ coerce source_class（显式优先 / source_type 映射）       guard.py:270-280
  │    ├─ [cls 冲突] emit HIGH BLOCK + raise ClassificationError   guard.py:295-320
  │    ├─ _pending_source_class = normalised（try/finally 复位）   guard.py:330
  │    ├─ _run_detectors（12 检测器 + exceeds_max_depth 追加）     guard.py:615-655
  │    ├─ _highest_severity / _decide（策略规则 → default）        guard.py:655-665
  │    ├─ BLOCK: emit + pre-block snapshot + raise PolicyViolation guard.py:340-355
  │    ├─ QUARANTINE: _quarantine[key]=value + emit                guard.py:355-365
  │    ├─ REDACT: _redact(value) + emit                            guard.py:365-375
  │    └─ ALLOW: store.set + note_independent_write + baseline     guard.py:375-400
  ├─ read(key, sink)                                               guard.py:470
  │    ├─ verify(key) → IntegrityError → CRITICAL BLOCK + raise   guard.py:480-490
  │    ├─ detectors → _decide → BLOCK raise / REDACT / ALLOW       guard.py:490-535
  ├─ delete(key)                                                   guard.py:540
  │    └─ ProtectedKeyDetector.matches → BLOCK raise；否则清 3 状态 guard.py:540-560
  ├─ promote(key, target, verified)                                guard.py:150
  │    └─ 晋升图 edge 检查 → requires_verification 门 → set+emit   guard.py:150-230
  └─ rollback(snapshot_id) → restore + emit                        guard.py:583-600
```

## 2. State Flow（状态流）
```
store keys ──(write ALLOW)──> committed value            InMemoryStore._data
classification: key → MemoryClass + origin_task          ClassificationRegistry.{_classes,_origin_task}
integrity:     key → SHA-256 digest                       IntegrityRegistry._baselines
quarantine:    key → value（BLOCK 改道，不入 store）        MemoryGuard._quarantine
self-reinf:    key → deque[(ts, value)] + last_indep      SelfReinforcementDetector._by_key
snapshots:     snapshot_id → Snapshot(data, digest)        SnapshotStore（label="pre-block"）
状态清除路径：delete → integrity.clear + classification.clear + reinf.reset（guard.py:560）
```

## 3. Data Flow（数据流）
```
write value
  → canonical_serialize（sort_keys/separators/default=str/ensure_ascii=False）
  → SHA-256 hash_value
  → baseline / verify（integrity.py:15-30）
  → _stringify（50 层截断 → TRUNCATION_MARKER）→ 12 detectors regex/语义扫描
  → exceeds_max_depth(value) → 追加 size_anomaly finding（guard.py:645-655）
  → _decide → action → store.set（可能被 redact 改写）
事件数据：SecurityEvent{detector,severity,action,key,message,source_class,receipt_uri,metadata}
  → to_dict() → handlers / OTel span（examples/opentelemetry_hook.py）
```

## 4. Evidence Flow（证据流）
```
SecurityEvent._events 列表累积（guard.events 只读视图）
  → _handlers 回调（add_event_handler 注册；handler 异常被吞，log.exception）guard.py:690-700
  → OTel：每事件一 span（attributes=SecurityEvent 字段）examples/opentelemetry_hook.py
  → receipt_uri：外部 Ed25519 共签审计链指针（events.py 文档；仓库无签名验证逻辑——指针仅记录）
  → 合规证据：compliance-mapping.md 所有数字解析到 benchmarks/results/*.json + benchmark_report.md
```

## 5. Authority Flow（权威流）
```
Policy.rules（有序）→ PolicyRule.applies_to(detector, severity, key) → action   policy.py:40-45
无匹配 → default_action（permissive=ALLOW / strict=ALLOW 但规则拦截）
决策升级：_decide 多 verdict → _escalate（ALLOW<REDACT<QUARANTINE<BLOCK）guard.py:660-680
信任权威：promote 需 PromotionRules.edge 存在 + requires_verification 必须 verified=True（用户 opt-in）
佐证权威：trusted_source_classes（默认 {SYSTEM}）决定 note_independent_write 是否衰减冷却
注：SourceClass 是调用方声明，不认证主体（events.py 明示）——权威流假设 provenance 诚实
```

## 6. Memory Flow（记忆流）
```
写记忆：write → 检测/策略 → store.set（或隔离/拦截）
读记忆：read → integrity.verify（基线不匹配 → CRITICAL BLOCK）→ 检测 → 交付
完整性生命周期：初始化对 immutable 键建基线（guard.py:115-120）
                   → write 提交后 immutable 键无基线则自动建（guard.py:395-400）
                   → delete 清除基线
快照记忆：snapshot_on_block=True → BLOCK 前 capture("pre-block")
           → rollback(snapshot_id) 恢复 store（digest 不验证——EK-20）
```

## 7. Policy Flow（策略流）
```
Policy 声明（YAML from_dict / strict() / tiered()）       policy.py:80-120
  → 决策点（_decide 逐 verdict 查 rules）                  guard.py:655-665
  → 执行（BLOCK raise / QUARANTINE / REDACT / ALLOW）      guard.py:340-400
  → 事件（每决策 emit，含 action 字段）                    guard.py:690-700
  → 未来决策受影响（快照 pre-block + rollback 允许恢复；self_reinforcement 历史衰减）
边界（tiered 文档明示）：TTL / revalidation / session lifetime / actor-role gating 未实现（roadmap）
```

## Flow→KO 交叉校验
| KO | 对应 Flow Edge（真实可回溯） |
|---|---|
| KO-01 | Control write 五阶段（guard.py:250-400）|
| KO-02 | Data exceeds_max_depth→size_anomaly（guard.py:645-655）+ Data _stringify 截断（injection.py:110-140）|
| KO-03 | Evidence detector 异常→LOW 事件（guard.py:620-640）|
| KO-04 | Data 12 检测器管线 + Memory 完整性基线 |
| KO-05 | Authority promote 门 + trusted_source_classes（guard.py:150-230；self_reinforcement.py:150-175）|
| KO-06 | Policy permissive 默认 + Memory digest 不验证（guard.py:52；snapshots.py:15-30）|
| KO-07 | Evidence SecurityEvent→handlers/OTel→compliance-mapping |
| KO-08 |（基准层，独立于运行时流）AMSB harness canary 判定 |
