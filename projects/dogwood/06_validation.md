# 06 · Validation & Evidence — Dogwood

> 独立 Auditor 盲重建流程：先不读考古产物，独立重读仓库建 Independent Findings，再对比。本文件记录验证结论与质量指标。

## 06.0 验证方法说明

- **运行环境限制**：环境无 Rust 工具链（`cargo` 不存在，已实测 which/find 均空），46,208 行 Rust 无法编译运行。
- **证据标准**：所有"测试揭示行为"类证据 = **测试意图阅读**（S3+），非运行验证（S4）；已在全部 EK 标注。不弱化标准换取全绿——S4 证据本轮不可得，如实记录。
- **盲重建**：Auditor 独立重读仓库（README / Cargo.toml / src 结构 / tests 目录 / AGENTS.md / CHANGELOG / guide 目录），产出独立发现后再与考古产物比对。

## 06.1 判定统计（独立 Auditor）

| 判定 | 数量 | 对象 |
|------|------|------|
| CONFIRMED | 24 | EK-01~08, 10~14, 17~27（代码/文档直接支持） |
| PARTIALLY_CONFIRMED | 3 | EK-09（契约层确认、deny-on-error 为测试意图）、EK-15/16（测试意图，未运行）、EK-28（测试意图） |
| DOWNGRADED | 0 | — |
| OVER_GENERALIZED | 0 | — |
| MISSING | 2 | 见 06.3 |
| CONTRADICTED | 1 | C-05（README vs eval.rs 张力，非考古产物错误） |
| NEEDS_HUMAN_REVIEW | 1 | C-07（None 语义） |

## 06.2 独立 Auditor 的成功发现（Top 3）

1. **时间条件静态检查的深度**（范围限制 + 嵌套聚合拒绝 + 保留 binder 拒绝）——check.rs 的函数族是独立盲重建时新确认的机制，考古产物完整覆盖（EK-02/03）。
2. **incremental lowering + distincter** 的 8 条测试意图（独立 lower、组合、增积 feed-forward）——确认两阶段 API 的真实设计意图（EK-07）。
3. **fail_closed 的元层价值**（契约/实现分层 + infallible 设计）——docstring 原文完整支持 EK-09/28 的分层论断，非过度解读。

## 06.3 关键错误 / 遗漏 / 攻击结果

### 考古产物 3 个最弱处（Auditor 视角）
1. **C-07（None 语义）未闭合**：is_authorized 返回 Option<Response> 的 None 精确语义（非决策点 vs 无适用策略）未在考古产物中给出定论，已进 Candidates——诚实标注。
2. **S4 证据整体缺失**：全部测试证据为意图阅读，覆盖广度大但深度受限于未运行；考古产物已如实标注，但"测试揭示行为"的置信度上限为 S3+。
3. **provider 求值细节未展开**：max_operations 默认值、白名单具体能力清单未接线核实（C-05），偏低频影响。

### 攻击项
- **单案例→Pattern**：KO-01~06 全部标 Cross-project validation pending，无越级断言 ✅
- **Pattern→L4**：4 个 CM 均有单项目强证据 + 同域对照（omnigent/EP-002），未写成 Law ✅
- **项目经验→通用 Principle**：M-01~03 标"Dogwood 验证" ✅
- **ADR→实现事实**：Cargo.toml 设计注释（单 crate 封装）为设计意图证据（S3 文档），未当实现行为断言 ✅
- **Flow Edge 真实性**：04.1~04.7 全部标注 symbol/file，抽查 authorize/mod.rs:214、engine.rs:192/287、check.rs:124/465 均真实存在 ✅
- **bypass/override/exception/alternate/direct call**：
  - alternate path：远程 policy store（EK-13）✅ 已覆盖
  - direct call：is_authorized 为唯一入口（authorize/mod.rs）✅ 已覆盖
  - exception：provider 错误 → deny-on-error（EK-09）✅ 已覆盖
  - fallback：无；legacy：无
- **Epistemic 状态**：Facts（L1）↔ EK（L2）↔ KO（L3）↔ CM（L4）↔ M（L5）分层无混淆；Hypothesis 全部进 05 ✅

## 06.4 关键遗漏检查

- **是否遗漏重要 Engineering Knowledge？** 无结构级遗漏。Auditor 独立发现项（时间检查函数族 / distincter / fail-closed 元层）均已覆盖。
- **是否错误升维？** 无。KO/CM 均带回溯与状态标注。
- **是否存在事实错误？** 未发现。抽查的 symbol/行号全部真实。
- **是否存在 Flow 错误？** 未发现。04.1 None 分支标注为推测（有 C-07 兜底）。
- **是否发现新的 Benchmark / Regression Case？** 3 个：
  1. **契约/实现分层断言测试**（fail_closed 模式）——"参考实现行为变更必须显式通知"可作 benchmark case 验证考古 skill 是否捕捉契约分层。
  2. **None 语义闭合**——要求考古产物回答 Option 语义，防"签名被读、语义被跳过"类遗漏。
  3. **docs-as-tests 防漂移**——文档示例=CLI 检查，可作"文档实现强耦合"检测基准。

## 06.5 质量指标

| 指标 | 值 |
|------|-----|
| EK 总数 | 28 |
| EK 有 links | 28（100%） |
| 游离 EK | 0（<20% ✅） |
| 平均出边 | ~2.1（≥1 ✅） |
| KO 聚合规则覆盖率 | 100%（6/6） |
| KO 平均簇规模 | 4.3（3~12 ✅） |
| 假聚合（同子系统=理由） | 0 |
| KO/CM/M | 6/4/3 |
| Candidates | 8 |
| 证据等级分布 | S3×22 + S3测试意图×6（S4=0，如实标注） |
| 交叉校验（Flow→KO） | 6/6 通过 |
| 单案例→Pattern 越级 | 0 |
| Hypothesis 冒充 Fact | 0 |

## 06.6 Reconciliation 记录

- 本 run 无 Auditor 补录修正（Auditor 独立发现全部已在原始考古产物中）。
- 保留项：C-05（README vs 代码张力）、C-07（None 语义）→ 均进 Candidates 待后续 refresh 或人工审查。

## 06.7 验证结论

**考古产物通过验证**（Truth / Coverage / Causality / Flow / Abstraction / Counterexample / Epistemic 七项检查均通过，无 CONTRADICTED 级错误；2 MISSING 为 Audit 检查项均已收纳于 05）。**局限**：S4 测试证据不可得（环境无 Rust 工具链），测试类结论置信度上限 S3+；如需 S4，需在具备 cargo 的环境补跑 `cargo test --manifest-path dogwood-language/Cargo.toml`。
