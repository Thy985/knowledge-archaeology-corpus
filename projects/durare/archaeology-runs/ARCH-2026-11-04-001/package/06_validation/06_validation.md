# 06 Validation & Evidence — durare

> 本文件 = 阶段④产出的 Validation 设计 + 自检 + 反例预算 + 证据索引。
> 阶段⑤将启动独立 Auditor **盲重建**（不把本包当事实来源，独立重读仓库）并产出 Independent Findings，对比结果经 Reconciliation 后合并入 corpus（最终版见 corpus 的 06-validation/validation-report.md）。

## 6.1 Blind Reconstruction 准备（阶段④内自检，非独立审计）

> 诚实声明：本节是 Synthesizer 侧的自检，**不替代**阶段⑤的独立 Auditor。盲重建由独立 Auditor 在阶段⑤执行。

## 6.2 反例预算制（每个 L3+ KO ≥3 定向攻击）

| KO | 反例攻击 | 结果 | 处理 |
|---|---|---|---|
| KO-01 位置即历史 | 官方 Rust SDK 容忍 spawned-task step（无 checkpoint 运行） | 幸存（contrast 明确） | 降级为"设计选择"表述，标注对照 |
| KO-01 | `patch`/`deprecate_patch` 仍在 poll 时分配（例外） | **击中** | 修正 claim：范围限定"大多数 durable call"，例外显式列出 |
| KO-01 | body 内构造 PendingStep 被拒绝但不占位 | 部分击中 | 表述改为"构造时 claim 或明确拒绝" |
| KO-02 两层编排 | single-engine 架构（如无 seam 的单体）也能工作 | 幸存（解释范围不扩大则降级） | 保留 L4 但标注"durare 验证的模型，非唯一模型" |
| KO-03 控制错误隔离 | 业务代码可 catch 一切（包括控制错误）后静默返回 | 幸存（latch 幂等拒绝机制防止） | 保留 |
| KO-03 | ObservationFailed 不调度恢复 → 若其实未发生会挂死 | **部分击中** | 降级表述：标注"at-least-once 语义下该选择避免重复副作用，代价是可能延迟恢复" |
| KO-04 DB 仲裁 | dequeue ENQUEUED→PENDING 条件更新失败时需重试而非错误 | 幸存（匹配 go/py） | 保留 |
| KO-04 | 事务序列化 vs 原子语句——Postgres 单语句本来就是事务 | 幸存（机制一致） | 表述精确化 |
| KO-05 事务原子 | read-only 事务退化为普通 checkpoint（window 仍在） | **击中** | 修正 claim：消除 window 仅对写事务；read-only 显式例外 |
| KO-06 字节协议 | 用户值含 `__DBOS_STREAM_CLOSED__` 字样会碰撞吗 | 幸存（经 serializer 编码不会 verbatim 碰撞，provider.rs 注释） | 保留 |
| KO-07 版本门控 | 固定 app_version 部署 0.4→0.5 不兼容 | **击中** | 修正 claim：版本门控安全的前提是版本反映语义，固定版本需先 drain |
| KO-08 确定性 | UnexpectedStep 事后 catch 可能漏掉未触发的分歧 | 幸存（backstop 定位明确） | 保留 |
| KO-09 SQL 不变量 | schema 标识符来自配置（用户可配 table name）——唯一例外路径 | 幸存（白名单正则 + 仅编译期常量） | 保留，标注例外 |
| KO-10 测试方法论 | SQLite 无条件跑 + Postgres 条件跑 → Postgres 专属 bug 可能漏 | **部分击中** | 标注测试矩阵缺口（DATABASE_URL 条件） |

**反例处理结果**：14 项攻击 → 3 项击中并修正 claim（KO-01 例外、KO-05 read-only、KO-07 固定版本）、2 项部分击中并降级表述（KO-03、KO-10）。修正后的 claim 见 03 层终稿。

## 6.3 Epistemic 状态自检（Fact/Observation/Hypothesis/Pattern/Model/Principle 不混淆）

| 检查项 | 判定 |
|---|---|
| Hypothesis 是否进 Candidates 而非知识层 | ✓（C-01~C-06 全部在 05） |
| 跨项目候选是否写成 Principle | ✓（C-06 显式标 Cross-project Hypothesis） |
| L4/L5 是否都有强证据（S4）支撑 | KO-02/03/04/06/08 为 S3+S4 混合，均标注 durare-originated · Cross-project validation pending |
| 文档声明（doc）与实现（code）是否区分 | ✓（EK-39 标注"文档 + 上游 commit 引用"；explicit-context.md 对照表为文档声明） |
| 测试断言是否存在（S4 判定是否属实） | ✓（S4 均标注"测试代码存在且断言如此，未经本环境运行"——证据上限声明） |
| 无"单案例→Pattern"跳跃 | ✓（每条 L3+ 至少 1 个反例或对照） |

## 6.4 Contradictions 与 Counterexamples（保留，不删除）

- **CONTRADICTION-1**: KO-01 宣称"position 构造时分配"与 EK-03 "body 内构造 PendingStep 不占位"表面矛盾——**已解决**：claim 范围 = 合法放置的调用在构造时 claim；被拒绝的放置（嵌套/跨执行）不占位是拒绝路径，非 claim 路径。
- **CONTRADICTION-2**: EK-12（read-only 退化普通 checkpoint）与 KO-05（事务消除 window）——**已解决**：KO-05 修正为"写事务消除；read-only 保留 at-least-once（无害）"。
- **COUNTEREXAMPLE-1**: 官方 Rust SDK 的容忍式放置（StepBuiltElsewhere）证明 durare 的严格拒绝是选择不是必然。
- **COUNTEREXAMPLE-2**: `patch`/`deprecate_patch` poll 时分配 position，证明"构造时 claim"不是普适定律而是默认纪律。

## 6.5 质量指标（本包）

| 指标 | 值 |
|---|---|
| EK 数 / 有 links / 游离 | 50 / 50 / 0 |
| EK 平均出边 | ≈2.1 |
| KO 数 / 聚合规则覆盖 | 10 / 100% |
| KO 平均簇规模 | 4.8 |
| Candidates | 6（4 Hypothesis + 1 安全敏感 + 1 Cross-project） |
| Flow 七类 | Control/State/Data/Evidence/Authority/Memory/Policy 各≥3 edge |
| 反例攻击数 | 14（3 修正 + 2 降级 + 9 幸存） |
| 证据上限 | S4（无运行时环境，benchmark 类自述 S3-S4） |

## 6.6 Evidence 索引（S4 测试证据）

| 测试文件 | 验证的 EK/KO | 断言要点 |
|---|---|---|
| tests/crash_consistency.rs | EK-24, KO-05, KO-10 | 7 边界精确 count（checkpointed step==1, effect-but-no-checkpoint==2, body==2） |
| tests/adversity.rs | EK-09, EK-20, KO-04 | fleet 竞争：exactly-once、dedup 唯一赢家、launch-recovery 不偷活 |
| tests/interop.rs | EK-27, KO-06 | golden strings byte-identical（pinned in py/ts/java/go suites） |
| tests/security.rs | EK-14, EK-28, KO-09 | EVIL sweep verbatim round-trip + 系统表完整 |
| tests/child_atomic.rs | EK-10, EK-25, KO-05 | contract(db) 三后端同跑（事务+child 原子） |
| tests/durable_call_nesting.rs | EK-03, KO-01 | 嵌套/跨 body 放置拒绝 |
| tests/step_cancellation.rs | EK-45 | Error::Cancelled 控制流 |
| tests/recovery_liveness.rs | EK-22 | 恢复不阻塞后续 run |
| tests/pg_client_debounces.rs | EK-42 | debounce（已知 flake #244） |

## 6.7 移交阶段⑤的审计重点

1. 重读 `src/execution.rs` 验证 control-error 通道与 latch 语义是否如实
2. 重读 `src/postgres.rs` claim_for_recovery / record_step_result / run_transaction_step 的 CAS 与 CTE 是否属实
3. 重读 `src/serialize.rs` 验证 format-directed 解码
4. 验证 EK-39 对照表是否与 upstream commit 7937413 的源码一致（文档声明 vs 实现事实）
5. 攻击 L4/L5 升维是否过度（KO-02/03/04/08 的"解释范围扩大"论证）
6. 检查 bypass/override/exception/alternate/direct call/admin path/fallback/legacy path：patch 例外、read-only 例外、`application_version=''` legacy 路径、warn_on_legacy_empty_app_version 已覆盖
