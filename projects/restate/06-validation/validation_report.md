# Validation & Evidence — ARCH-2026-10-27-001

## 六审计（主考古自检）
| 审计 | 结果 | 说明 |
|---|---|---|
| Truth | PASS_WITH_RESERVATIONS | 所有 EK/KO/Flow 均有 [code] 锚（文件:行）；reservation：replicated_loglet 复制协议细节（sequencer 故障转移）未深读 |
| Coverage | PARTIAL | 覆盖 6/8 核心面（journal/processor/partition-store/bifrost/vqueues/invoker）；未精读：metadata-server 全实现、query_engine(DataFusion)、object-store-plugin、cloud-tunnel-client、admin API |
| Flow | PASS | 22 Edge 全可回溯 symbol/file；7 类流完整；3 处 [code] 锚为目录级（migrate_journal_table/C-R04） |
| Abstraction | PASS_WITH_PENDING | 8 KO 中 7 条标 Cross-project validation pending；KO-R06 项目内已验证（三 provider 并存） |
| Counterexample | PASS | 反例：EK-R14（suspend 前 notification 检查——"Resuming instead of suspending"）；EK-R05（retention=0 不去重）；EK-R10（journal ≤ completion 强制） |
| Epistemic | PASS | 无 Hypothesis 冒充 Fact；C-R01..07 全标假说；C-R07 显式已排除 |

## 质量指标
- 采证锚：>40（proto/code/doc/test 带文件定位）
- EK：24 条（平均出边 2.0，游离 0，聚合覆盖率 100%）
- KO：8 条（L3×3/L4×4/L5×1；7 条 pending）
- Flow：7 类 22 Edge（[code] 锚为主，3 处目录级 [pend]）
- Candidates：7（跨项目假说 3 / 暂定 3 / 已排除 1）

## 待 IVR 攻击面（盲重建重点）
1. journal Command 20 种枚举与 completion 关联（command.rs）
2. is_resumable 语义与重放跳过（journal_table）
3. vqueues 串行化 + lock_table 协同（scheduler/locks）
4. bifrost append→Lsn 确认点（appender/bifrost.rs）
5. OnMaxAttempts Pause|Kill 语义（schema/invocation_target.rs）
6. dedup_table 生产者去重（deduplication_table）
7. suspend/resume notification 优先（suspend.rs:80）
8. RuleBook 日志一致（upsert_rule_book.rs）
9. BUSL-1.1 license 条款（LICENSE）
10. ServiceInvocation 幂等/retention 语义（invocation/mod.rs）
