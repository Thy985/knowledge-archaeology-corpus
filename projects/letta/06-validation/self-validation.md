# Self-Validation（生产侧，非独立审计）

> 本节由考古执行方自检，**不能替代阶段⑤的独立 Auditor 盲重建**。

## 1. Truth（事实）

| 检查 | 结果 |
|---|---|
| 每条 EK 是否至少有 1 条 E 编号 | 通过（20/20） |
| 每个 E 是否有 file:line | 通过（40/40，个别为目录级如 E-35/37） |
| 是否有"文档声称"被当作实现事实 | 已隔离：AGENTS.md/SECURITY.md 只进 Policy Flow（P-01~03），不进 EK |
| 数字来源 | 1156/878/248417/166/95/31/167 均由 `os.walk` + 计数脚本产出；模块行数为脚本 rollup |

**薄弱点**：E-28（archive 检索）与 E-30（compact 阈值）为**部分阅读**，只读到函数签名与关键分支，未逐行确认完整调用链。

## 2. Coverage（覆盖）

已覆盖子系统：memory schema、block ORM/history/pivot、block_manager（DB）、block_manager_git、passage_manager（写入/双写）、summarizer（两策略）、tool_executor（memory 工具）、auth、ORM 租户 mixin、AGENTS/SECURITY。

**未覆盖（明确缺口）**：
- `letta/managers/` 22,928 行（agent/block/passage 之上的编排）——可能含额外的权限/批处理逻辑。
- `server/rest_api/routers/v1` 30 个 router 的鉴权装饰器实际使用情况。
- `llm_api` / `local_llm` provider 实现。
- 167 个迁移的完整演进（仅按文件名推断）。
- `agents/` 中 v1/v2/v3 的执行循环差异（只引用未细读）。

## 3. Flow（流）

- 28 条边均给出 symbol/file；七类齐全（Control 5 / State 4 / Data 4 / Evidence 3 / Authority 4 / Memory 5 / Policy 3）。
- 失败路径已单列：C-05 旁路、D-03 双写、D-04 摘要丢失、V-03 审计缺口。

**薄弱点**：C-01（agent loop → executor）只给出目录级证据，未逐行定位 `letta_agent_v3` 的分派点；Flow Edge 的真实性需独立审计复核。

## 4. Abstraction（抽象）

- 最高 L3；**0 条 L4/L5**。
- KO-05 主动降为 L2 Observation（单点证据、无反例）。
- KO-04 标注"弱 L3"并声明"从字段不能表达推出，未观察到实际越权"。

**自检结论**：未发现"单一案例 → Pattern"的跳跃，但 KO-04 的抽象强度依赖 CAND-06 是否成立。

## 5. Counterexample（反例）

| KO | 反例数 | 是否达标（≥3） |
|---|---|---|
| KO-01 | 3 | 是 |
| KO-02 | 3 | 是 |
| KO-03 | 3 | 是 |
| KO-04 | 3 | 是 |
| KO-06 | 3 | 是 |

反例均为"定向搜索到的替代解释"，不是泛泛的"可能不成立"。

## 6. Epistemic（认知状态）

| 类型 | 数量 | 说明 |
|---|---|---|
| Fact（E） | 40 | 全部 S3 |
| EK（L1/L2） | 20 | 全部"已实现" |
| KO-L3 | 5 | 全部 `Cross-project validation pending` |
| KO-L2 | 1 | Observation |
| Candidate | 6 | Hypothesis，明确禁止升格 |
| Principle/Law | 0 | — |

**未发现**：Hypothesis 被写成 Fact；Cross-project Candidate 被写成已验证 Principle。

## 7. 自检不通过项（如实记录）

1. **运行验证为 0**：所有结论上限 S3，不能称"已验证行为"。
2. **KO 数量 6 < 建议 7~12**：按"不为凑数"原则保留，需在报告中说明。
3. **CAND-01 未决**：legacy 工具可达性直接决定 KO-03 的强度，本轮未证实。
4. **Flow Edge 抽样未复核**：无独立 flow auditor，C-01 等目录级证据需下游复核。
