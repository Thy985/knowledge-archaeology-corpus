# 06 Validation — Self Validation（生成侧自检）

> 说明：这是**生产侧**自检，不是独立验证。独立盲重建见 `validation/independent_validation.md`；两者结论分开记录。

## 1. Truth Auditor（真实性）

抽样复核（从产物中随机取 12 条，回到仓库原文核对）：

| 断言 | 回读位置 | 结果 |
|---|---|---|
| 抽象类只有 3 个抽象方法 | `EvolveLab/base_memory.py:10-51` | CONFIRMED |
| `MemoryStatus` 仅 BEGIN/IN | `memory_types.py:11-15` | CONFIRMED |
| 13 项 PROVIDER_MAPPING | `memory_types.py:48-63` | CONFIRMED |
| 记忆以 user 消息注入 planning | `agents.py:413-425` | CONFIRMED |
| 每 step 注入 IN | `agents.py:782-795` | CONFIRMED |
| 7 个 provider 仅 BEGIN | 7 个文件分支行（E-021） | CONFIRMED |
| Lightweight 长期默认关闭 | `lightweight:108-116` + `config.py:44-53` | CONFIRMED |
| Validator 默认 auto-fix | `phase_validator.py:29-43` | CONFIRMED（与文档矛盾） |
| Creator 写入顺序 | `memory_creator.py:86-284` | CONFIRMED |
| Evolver 不存 embeddings | `evolver:161-175` vs `123-159` | CONFIRMED |
| MemP 缺失默认成功 | `memp:480` | CONFIRMED |
| 核心模块无测试 | `find` 全仓 | CONFIRMED |

**失败项**：无（12/12 与源码一致）。

## 2. Coverage Auditor（覆盖度）

| 子系统 | 覆盖情况 | 缺口 |
|---|---|---|
| `EvolveLab` 接口/类型 | 完整 | — |
| 13 providers | 每个至少核了阶段分支 + 存储路径；6 个做了机制级细读（Agent-KB/Cerebra/Lightweight/Evolver/MemP/Dynamic） | Dilu/Generative/Voyager/MobileE/Expel 只做了关键词级核对，未逐行读写入细节 |
| `MemEvolve` 四阶段 | Analyze/Generate/Create/Validate 主线完整 | `ReviewAgent`、`evolve_cli` 全部子命令未逐行 |
| `AutoEvolver` | 公平性、选择、复评、checkpoint 已核 | 失败重试/超时/异常恢复路径未读全 |
| `FlashOAgents` | 两个注入点完整 | 其他 Agent 类型（MMSearchAgent 等）未核 |
| `mini-swe-agent` | 只确认 tests 存在 | 未读其实现（vendored，非本项目核心） |
| `model_eval` | 未覆盖 | 缺口 |
| 运行验证 | 无 | 无 API key/数据，S4/S5 缺失 |

**覆盖结论**：核心链路覆盖充分；外围（model_eval、vendored、部分 provider 细节）存在缺口，已在每条结论的 scope 中限制。

## 3. Flow Auditor（流一致性）

- 七类流均存在，共 **68 条 Edge**（F1 15 / F2 12 / F3 10 / F4 8 / F5 7 / F6 9 / F7 7）。
- 抽查 10 条 Edge 的 symbol/file/condition：全部可在源码定位（含 3 条 fail、4 条 alt、3 条 bypass）。
- 每条 KO 均标注依赖的 Flow Edge（9/9）。
- 发现的“无流可依”断言：0 条进入 KO；2 条降级到 Candidates（C-02 并发、C-06 工具执行）。

## 4. Abstraction Auditor（抽象层级）

| KO | 申报层级 | 自检结论 | 理由 |
|---|---|---|---|
| KO-01~05, KO-08, KO-09 | L3 | 保持 | 有 ≥2 独立子系统/文件对照，解释范围扩大 |
| KO-06 | L4 | 保持但标注 pending | 机制链完整（7 EK），但仅单项目证据 |
| KO-07 | L4 | **降级风险**：本应 L3+候选 | 仅 1 个项目、1 个验证器；已保留 L4 但明确 pending，交由独立审计复核 |
| P-01~P-16 | L3 候选 | 不升 KO | 单实现/缺对照 |

**未发现**“单案例→Pattern”或“项目经验→通用 Principle”的未标注升维。

## 5. Counterexample Hunter（反例预算）

- L3+ KO 共 9 个，每个 3 条定向反例 = **27 条**。
- 0 反例的 KO：无。
- 反例中有 6 条实质削弱了 KO 的适用范围（KO-01 A/C、KO-02 A/B、KO-03 C、KO-09 C），已在 Scope 中收紧。
- 未发现“反例推翻 KO”的情况；KO-06 的反例 C 仅削弱影响范围。

## 6. Epistemic Auditor（认知状态）

| 检查 | 结果 |
|---|---|
| 论文数字是否被写成 Fact | 否（E-092~E-096 全部 S0-vendor；C-07 标 Hypothesis） |
| 单项目模式是否被写成 Principle/Law | 否（KO-06/07 标 Model + pending） |
| 候选是否混入 KO | 否（C-01~C-14 独立目录） |
| 文档声明是否被当实现事实 | 否（README 的 11+2 映射列为 C-05） |
| 未验证缺陷是否被写成事实 | 否（C-01/C-02/C-04 均 Hypothesis） |

## 7. 质量指标（自检侧）

| 指标 | 值 | 目标 | 判定 |
|---|---|---|---|
| Evidence 条数 | 100 | 100+ | 达标（边界） |
| S3 证据占比 | 96% | — | 强 |
| EK 条数 | 50 | 40~60 | 达标 |
| EK 平均出度 | 2.68 | ≥1 | 达标 |
| 游离 EK | 0% | <20% | 达标 |
| KO 数 | 9 | 7~12 | 达标 |
| KO 平均簇规模 | 4.67 | 3~12 | 达标 |
| aggregation_rule 覆盖 | 100% | 100% | 达标 |
| L3+ 反例数 | 27 | ≥3/KO | 达标 |
| Flow Edge 数 | 68 | — | 七类齐全 |
| 运行证据 | 0（S4/S5） | — | **缺口**（无 API key/数据） |

## 8. 自检结论
`PARTIALLY_VALIDATED`：静态事实与机制链完整、认知状态标注诚实；但缺少运行验证（S4/S5），且外围模块有覆盖缺口。独立验证需重点攻击 KO-07 的层级与“13 provider 差异”的概括范围。
