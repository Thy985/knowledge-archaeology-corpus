# Independent Validation Report — ARCH-2026-09-16-001

> 独立 Auditor 盲重建：不把 Archaeology Result 当事实来源，独立重读仓库关键路径后对比。

## 判定统计

| 判定 | 数量 | 条目 |
|---|---|---|
| CONFIRMED | 21 | E-01~E-14 主体、E-15/16、E-19/20 |
| PARTIALLY_CONFIRMED | 3 | EK-13（ACS 确定性：仓库内是 shim）、EK-33（无 LLM 表述过窄）、KO-02（表述需限定） |
| DOWNGRADED | 1 | KO-02：L4 表述从"无模型在决策路径"收紧为"确定性层为规范信任边界" |
| OVER_GENERALIZED | 1 | KO-06："治理栈从 OS 借力"需限定为"AGT 是参考实现，非品类必然" |
| MISSING | 2 | advisory 层、authority resolver（第二决策路径） |
| CONTRADICTED | 1 | EK-08 "Merkle 哈希链" vs 实现"哈希链 + Merkle 树双结构" |
| NEEDS_HUMAN_REVIEW | 2 | C-09（外部 crate 版本一致性）、E-17 vs E-18（13,834 vs 992 缺口） |

## 3 个最重要的成功

1. **决策管线顺序完全确认**（govern.py:231-330 盲读）：ring → policy → approval → audit → advisory → execute 的固定顺序与 Ring 注释（"runs before policy evaluation so a denied ring never reaches the policy engine"）逐字吻合。分层防御是真实代码顺序，非架构叙述。
2. **fail-closed 无开关确认**（policy.py:168-172 + ADR-0013）：allow 规则失败 → match=False（fail-open 危险被显式注释），其余规则失败 → 匹配失败不放过；ADR-0013 明确"无配置开关切 fail-open"。这是治理正确性的核心支柱。
3. **审计可证明性三层确认**（audit.py:246-330 + decision_bom.py + trace_sink.py）：AuditEntry.previous_hash 链（verify_chain 校验链完整性）+ Merkle 树根哈希（verify_proof）+ DecisionBOM 四源重构 + TRACE 信任记录——四层证据机制全部真实存在于代码。

## 3 个最重要的错误

1. **【MISSING + CONTRADICTED】advisory 层完全遗漏**：GovernedCallable.__call__ 在确定性 allow 之后、执行之前还有 **advisory 层**（advisory.py）：分类器式 defense-in-depth，只能收紧（block/flag_for_review），不能放宽；失败默认 allow（"deterministic layer is canonical"）；决策带 `deterministic: false` 审计标记；含 SSRF 防护（_BLOCKED_HOSTS：cloud metadata 169.254.169.254 等）。原 EK 把 AGT 描述为"纯确定性"，这是错误——**确定性是规范层，advisory 是非确定性可选层**。
2. **【MISSING】authority resolver 第二决策路径遗漏**：policy.py:evaluate 尾部在"无 YAML 规则匹配"时走 authority_resolver（trust-based narrowing）——DelegationInfo/TrustInfo/ActionRequest 组合的信任窄化路径。这是"YAML 策略未覆盖时的兜底授权面"，原 EK 只写了一条决策路径。
3. **【CONTRADICTED】Merkle "链" vs "树"表述不准**：audit.py 实现是**哈希链（previous_hash 连续性，verify_chain）+ Merkle 树（增量构建根哈希，verify_proof）双结构**。ADR-0017 文字称 "hash chain (Merkle chain)"，但代码同时维护 _tree。原 EK-08 只写链，未写树。

## 关键遗漏？
- **有**：advisory 层、authority resolver（上面两处）。两者都影响"决策路径有哪些面"的完整性。

## 错误升维？
- **KO-02 需降级**："无 LLM 在决策路径"原文来自 README/SECURITY 的自述，但盲重建证明存在非确定性 advisory 层（虽然它只能收紧不能放宽，且失败默认 allow）。KO-02 表述"可被模型影响的决策不是治理"过强——修正为：**核心授权决策确定性（规范层），附加防线可非确定性但无授权能力（advisory 只能 block 不能 allow）**。这反而是更强的模式：**非确定性层被剥夺了"放行权"，只有"阻止权"**——这是比"无模型"更精确、更可迁移的认知模型。
- **KO-06 过泛**：AGT 确实是"Agent 的 Linux"全栈参考实现，但 Guardian/AI Protector 未用执行环/SRE 词汇表——"治理栈从 OS 借力"是 AGT 的设计选择，不是品类共性。降级为"AGT 采用 OS 治理词汇表，品类内是孤例参考实现"。

## 事实错误？
- 无硬事实错误（file:line 引用均存在）。E-18（992/29）已诚实标 S7。

## Flow 错误？
- **Control Flow 需补 advisory 边**：`decision.allowed → advisory → block? → raise / execute`（govern.py:310-323）。原 Flow Atlas 缺此边。
- **Data Flow 需补 authority resolver 边**：无 YAML 匹配 → DelegationInfo/TrustInfo/ActionRequest → 信任窄化。

## 新发现（盲重建独有）
1. **advisory SSRF 防护**：_BLOCKED_HOSTS（cloud metadata IP 黑名单）+ URL scheme 校验（仅 http/https）——这是原考古未发现的机制。
2. **authority resolver 输入面**：trust_score 默认 500 / risk_level 默认 medium 进 DecisionBOM 的 TrustSource——默认值影响决策。
3. **DENIED_COMMANDS 精细化**：curl/wget/nc/gcc/bash/sh/dash/perl/ruby/python2 等——命令黑名单按"exfiltration/C2/任意代码/生成可执行文件/子 shell 绕过"分类注释，可作 Benchmark case。

## 新 Benchmark / Regression Case？
- **B-05（新增）**：advisory 层"只能收紧不能放宽"契约测试——block 后不允许 allow 覆盖；失败默认 allow 且审计 deterministic:false。
- **B-06（新增）**：authority resolver 兜底路径测试——无 YAML 规则匹配时 trust narrowing 生效。
- **B-07（新增）**：SSRF 防护测试——metadata IP 域名/URL 拒绝。
- **B-03 强化**：trust ceiling 单调性 + authority resolver 协同测试。

## NEEDS_HUMAN_REVIEW
- **C-09**：policy-engine/core 是 shim，真实决策逻辑在 crates.io agent_control_spec——外部 crate 版本与本地 spec 的一致性需 Owner 或上游确认。
- **E-17 vs E-18**：13,834 test fn（代码扫描）与 992 conformance（README 自述）的精确归属未核实——建议跑 policy-engine conformance 套件。

## 结论
**Archaeology Result 主体可靠（21 CONFIRMED），但有 2 处实质性遗漏（advisory/authority resolver）+ 1 处表述需降级（KO-02）+ 1 处技术细节修正（Merkle 链+树）。** 全部修正进入 Reconciliation，不修改原考古产物。
