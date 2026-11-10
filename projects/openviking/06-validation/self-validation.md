# 06 · Self-Validation（考古方内部校验）

> 这是**考古侧自校验**，不是 Independent Validation。它只回答"我方是否把证据用对"，不回答"结论是否真的成立"。
> 独立 Auditor 的盲重建见 `independent_validation.md`。

## 1. Truth 抽样重读（3 条）

| 抽样 | 原 claim | 重读位置 | 结果 |
|---|---|---|---|
| T-01 | `rm` 先删向量索引、再删文件 | `openviking/storage/viking_fs/_ops.py` `rm()` 实现段：`vector_store.delete_uris(...)` 在两处分支（recursive / single）中均位于 `_async_agfs.rm` 之前 | **CONFIRMED**（code） |
| T-02 | 默认检索 tier 为 abstract，仅 events 可 full | `retrieve/context_assembler/params.py:56-82`：`DEFAULT_TIER="abstract"`；`DEPTH_CEILING_BY_CATEGORY={"events":"full"}` | **CONFIRMED**（code） |
| T-03 | trusted 模式必须 root key | `server/auth/plugins/trusted.py:139-151`：缺失/不符分别抛 `UnauthenticatedError`；`hmac.compare_digest` | **CONFIRMED**（code） |

## 2. Coverage 校验

### 已覆盖（有代码级证据）

| 子系统 | 覆盖度 | 证据 |
|---|---|---|
| VikingFS 删除/移动 | 高（核心路径） | `_ops.py` rm 实现；mv 仅文档 |
| 记忆写入 | 高 | `memory_updater.py` 全链 |
| 记忆 schema | 高 | 13 个 YAML + merge_op |
| 检索 | 中高 | retriever / intent / tiers / params |
| 会话提交 | 中 | 文档 + 队列类；Phase1 代码未逐行 |
| 权限 | 中高 | trusted plugin 全读；ACL 主要依赖文档 |
| 隐私 | 中 | 文档 + 目录结构；sanitizer/restorer 只读类定义 |

### 盲区（明确声明）

| 盲区 | 影响 | 后果 |
|---|---|---|
| `crates/ragfs`（130k 行 Rust） | 锁的原子性、崩溃恢复、租约语义 | 所有锁结论 ≤ S2-doc，标 `Rust-unread` |
| `openviking_gateway/`（50 文件） | 多实例路由、网关层身份透传 | Authority Flow 只覆盖直连 server |
| embedding/semantic 队列消费者 | 索引延迟、失败重试 | KO-01 的"索引落后"窗口无法量化 |
| `bot/`、`frontend/`、`web-studio/` | UI 侧策略 | 不影响核心结论 |
| 运行时 | 全部结论 ≤ S3；无 S4/S5 | 性能与并发行为未验证 |

### Coverage 结论

> **本 Package 覆盖"核心机制级"，不覆盖"全仓级"**。对 733k 行 Python 的仓库，声称"已看清全貌"是不诚实的；任何"OpenViking 总是…"的表述都超出了检查范围。

## 3. Flow 校验

| 检查 | 结果 |
|---|---|
| 每个 KO 至少依赖 3 个 Edge | 通过（最少 KO-04 依赖 5 条） |
| Edge 有 symbol/file/condition | 74 条中 59 条 code 支撑；15 条 doc-only 已标注 |
| Flow 与 KO 矛盾 | 1 处张力：EK-23（静默失败）vs EK-24（硬失败）——已在 KO-07 显式记录为"内部张力"，未强行统一 |
| 未验证 Edge 混入 Fact | 无；doc-only 全部标 `doc` |

## 4. Abstraction 校验

| KO | 宣称层级 | 校验 | 结论 |
|---|---|---|---|
| KO-01 | L4 | 5 条 EK 跨导入/删除/移动/向量化 4 条路径汇聚同一不变量 | 保留 L4，标 cross-project pending |
| KO-02 | L4 | 6 条守卫 + 1 条未验证边界（C-01） | 保留 L4，但 scope 收窄为"memory 写入路径 + 已验证的资源路径" |
| KO-06 | L4 | 5 条 EK 跨 auth/ACL/配置 | 保留 L4 |
| KO-03/04/05/07/08 | L3 | 均只有单项目证据；KO-07 含内部张力 | 保留 L3，不上 L4 |
| 是否有 L5 | 无 | 单项目不足以给"如何行动"的普遍准则 | 正确 |

## 5. Counterexample 统计

| KO | 反例数 | 是否找到真反例 |
|---|---:|---|
| KO-01 | 3 | 否（3 条均符合） |
| KO-02 | 3 | 1 条待验证（C-01） |
| KO-03 | 3 | 否（均为边界而非反例） |
| KO-04 | 3 | 否（schema 非封闭，但模式仍成立） |
| KO-05 | 3 | 1 条待验证（C-02 幂等） |
| KO-06 | 3 | 否 |
| KO-07 | 3 | **2 条真反例**（静默失败不可观测；collection 缺失静默） |
| KO-08 | 3 | 否 |
| **合计** | **24** | 真反例 2 条（KO-07） |

## 6. Epistemic 校验

| 检查 | 结果 |
|---|---|
| 是否有 Hypothesis 被写成 Fact | 无（Candidates 单独存放） |
| 是否有 Pattern 被写成 Principle/Law | KO-06 标 Principle 但带 `Cross-project validation pending`；其余为 Pattern |
| 厂商自报是否进入 Fact | 无；EV-65 标 S0-vendor，未进 EK/KO |
| 文档声明是否冒充实现事实 | 15 条 doc-only Edge 已标注；EV-16/39/40/42-45/51 标 S2-doc |

## 7. 数量纪律核对

| 项 | 实际 | 目标 | 判定 |
|---|---:|---|---|
| Facts/Evidence | 65 | 100+ | 低于目标（已保留原始粒度，不硬凑） |
| EK | 36 | 40–60 | 略低（记录，不补凑） |
| Patterns/KO | 8 | 7–12 | 通过 |
| L4 | 3 | — | 通过 |
| 平均出边 | 2.72 | ≥1 | 通过 |
| 游离 EK | 0% | <20% | 通过 |
| KO aggregation_rule | 100% | 100% | 通过 |

> 未为凑数制造知识点：Evidence 65 < 100、EK 36 < 40 均如实记录。

## 8. 自校验结论

- **Pass with declared gaps**：核心机制链（存储/写入/检索/权限）证据闭合；Rust 锁、网关、队列消费者为已知盲区。
- 不将"文档写了"等同于"代码做了"：15 条 Edge 明确标 doc。
- 不为全绿而弱化标准：KO-07 保留 2 条真反例并下调 value 至 B；KO-02 保留未验证边界。
