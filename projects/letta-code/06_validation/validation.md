# 06 Validation & Evidence

> 六 Auditor 先 Blind Reconstruction（独立重读仓库后对比），再判定。本文件是阶段 4 自检；阶段 5 独立 Auditor 报告见 `independent-audit.md`。

## 1. Truth Auditor（Source Truth）
**方法**: 对每条 KO/EK 关键 claim，回仓库找符号/文件/条件。
**结果**: PASS
- 抽查 25 条 claim，24 条直接命中源码符号/注释；1 条部分命中（F-15 内置 skills 数量=19，来自 `ls src/skills/builtin/` 实测 19 目录）。
- **未发现事实错误**。
- 特别注意：`{CORE_MEMORY}` 注入在 system-prompt-compilation.ts 的 `injectCoreMemory` 中确认（含"若 prompt 无变量则追加"逻辑）。

## 2. Coverage Auditor（Coverage）
**方法**: 独立重搜记忆/治理/生命周期子系统，对照考古覆盖。
**结果**: PASS_WITH_GAPS
- 覆盖：记忆（MemFS/git/frontmatter/约束/子代理/投影）、权限（四模式/沙箱/路径墙/只读）、mods、skills、生命周期（headless）、CI。
- **GAP-1**: `src/backend/api/memfs-git-proxy.ts` 只确认了存在与"localhost 代理 URL 不持久化"注释，未读完整代理实现（credential 如何注入 git remote）。
- **GAP-2**: `bwrap.ts`/`seatbelt.ts` 的具体沙箱参数（mount namespace 配置）未完整阅读——C-02 依赖此。
- **GAP-3**: mods 的运行时装配（mod 如何被加载/执行/沙箱）未深入——C-08 依赖此。

## 3. Causality Auditor（Causality）
**方法**: 检查因果链 claim（reason→commit→hook 等）是否真实。
**结果**: PASS
- EK-06→EK-02→EK-03 因果链成立：memory() 强制 reason（tools/impl/memory.ts:103）→ applyMemoryCommand 写文件 → commitMemoryWrite（git commit 触发 pre-commit hook）。
- KO-08 因果链（存储→投影→自编辑→提交→门禁→迁移）每步均有源码。

## 4. Flow Auditor（Flow）
**方法**: 逐条检查 Flow Edge 真实性。
**结果**: PASS
- Control/State/Data/Evidence/Authority/Memory/Policy 七类流关键 Edge 全部回溯到符号。
- **Data Flow 关键发现**: `collectCommittedMemoryFiles` 用 `git ls-tree HEAD` + `git show HEAD:<path>`——**投影只读已提交内容**（非工作树），该 Edge 已在 flows.md 标注为设计意图。
- Memory Flow 全链（初始化→写入→门禁→提交→同步→检索→演化→迁移）每条 edge 有证据。

## 5. Abstraction Auditor（Abstraction）
**方法**: 检查 L3/L4/L5 是否过度升维。
**结果**: PASS（2 项降级建议已吸收）
- KO-02/KO-07 升 L4 通过（一句话稳定关系可述）。
- KO-08 升 L5 有条件通过：**标注 cross-project validation pending**（单项目证据，不冒充普适方法论）。
- **降级-1**: 原草案"记忆=文件是 agent 记忆终局"（L4 断言）→ 降为 C-01 Hypothesis（跨项目证据不足）。
- **降级-2**: 原草案"fail-closed 是唯一正确治理"（L4）→ 降为 KO-02 的对照叙事（交互路径 opt-in 是有意设计，非缺陷）。

## 6. Counterexample Hunter（反例预算）
**方法**: 每个 L3+ KO ≥3 个定向反例攻击。
**结果**: PASS（27 反例，9 KO × 3）
- 每个 KO 的反例在 ko.md 中列出。
- 额外攻击：`LETTA_FS_SANDBOX=0` 退出开关（KO-02 反例）；DEFAULT_PERMISSION_MODE=unrestricted（KO-07 反例）；memory 工具外直接编辑文件路径（KO-04 反例）。

## 7. Epistemic Auditor（Epistemic Status）
**方法**: 检查 Fact/Observation/Hypothesis/Pattern/Model/Principle 是否混淆。
**结果**: PASS
- KO-01/03/04/05/06/09 = Pattern（L3，可复用模板）——与 Epistemic 匹配。
- KO-02/07 = Cognitive Model（L4）——匹配。
- KO-08 = Methodology（L5，单源 pending）——已标注。
- 所有未验证假设在 C-01~C-08（Hypothesis），未混入 KO。
- **无 Hypothesis 冒充 Fact；无 Cross-project Candidate 冒充 Principle。**

## 质量指标
| 指标 | 值 | 门槛 | 状态 |
|---|---|---|---|
| Facts | 121 | ≥100 | ✅ |
| EK | 46 | 40~60 | ✅ |
| EK 边 | 64（avg 1.39/节点） | ≥1 | ✅ |
| 游离 EK | 0 | <20% | ✅ |
| KO | 9 | 7~12 | ✅ |
| 聚合规则覆盖率 | 9/9（100%） | 100% | ✅ |
| 假聚合（同子系统=理由） | 0 | 0 | ✅ |
| 反例 | 27（3/KO） | ≥3/KO | ✅ |
| Flow | 7 类 | 7 | ✅ |
| KO↔Flow 交叉校验 | 9/9 一致 | 全一致 | ✅ |
| Candidates | 8 | — | ✅ |
| 降级 | 2（草案→定稿已吸收） | — | ✅ |
