# Independent Validation Report — ARCH-2026-09-13-001（aigis）

> Auditor 纪律：不引用考古包结论为事实；独立重读仓库（aigis/ 源码 + tests/ + ARCHITECTURE.md）建立 Independent Findings 后再对比。

## 判定统计

| 判定 | 数量 | 对象 |
|---|---|---|
| CONFIRMED | 9 | 零依赖 / HMAC 链 / L1-L7 管线 / PolicyDSL 结构 / MCP trust score / Incident 生命周期 / Log 3-tier / 治理文档 / 测试 1711+1 |
| PARTIALLY_CONFIRMED | 2 | README 基准（93.5%/0.0% FP，项目自述）；fork 合规能力（53 项映射声明） |
| DOWNGRADED | 0 | — |
| OVER_GENERALIZED | 0 | — |
| MISSING（Auditor 补充） | 3 | fail-closed 语义（EK-17）；_regex_guard 用户规则防护（EK-18）；谓词取值映射（并入 EK-09） |
| CONTRADICTED | 0 | — |
| NEEDS_HUMAN_REVIEW | 1 | 测试失败（test_i18n LC_ALL）归因需维护者确认测试意图（LANG 应胜还是 LC_ALL 应胜） |

## 原 Archaeology 最重要的 3 个成功

1. **审计链防篡改的定位准确**（KO-03）：HMAC + hash chain + frozen entry + test_tampered_* 族四件套逐行核实——这是"证据不可抵赖"的完整实现，未被低估。
2. **诚实边界声明被保留为知识而非美化**（EK-15）：10 个 miss 类别"not claimed as solved"是安全工具罕见的自我约束，包正确标注为项目自述而非实证。
3. **测试失败归因正确**：1711/1 的失败确认为测试环境耦合（LC_ALL 未 mock），且 spy 实验验证实现符合 POSIX 标准——没有为了"全绿"而弱化证据。

## 最重要的 3 个错误 / 缺口

1. **MISSING：fail-closed 异常语义未被写入 EK**（claude_code.py:93-111——scan 异常→exit(2) 阻断、policy 异常→deny、仅日志异常→pass）。这是安全工具"异常时默认拒绝"的关键设计事实，比不少已写 EK 更重要。（已 reconciliation 补 EK-17）
2. **MISSING：_regex_guard 用户自定义规则防护**（防 re.error 崩溃 / 防吞异常静默禁用 / 防 ReDoS）——"用户扩展面本身被加固"是安全工具的独特设计，包内未提。（已 reconciliation 补 EK-18）
3. **PARTIAL：KO-02 正交分层反例标注略轻**——L4 taint 与 L5 sandbox 的功能重叠（都拦"危险执行"）已在包内标注但未展开"重叠是否导致绕过"（重叠层的逃逸路径差异未验证）。（保留为反例证据，不升维）

## 是否存在关键遗漏？
否（核心机制全覆盖）。补充项为增强非修正：fail-closed 语义与用户规则防护属于"策略流/边界"的补强。

## 是否存在错误升维？
否。4 个 KO 均 L3/L4 且有 ≥2 EK 簇内支撑；L5 方法论被正确扣留（单项目证据不足）。

## 是否存在事实错误？
否。抽查零依赖（pyproject）、HMAC canonical（signed_log.py:274-275）、verify_chain genesis（chain.py:45-46 "0"*64）、trust 分级（_TRUST_LEVELS untrusted=0 默认）全部与包内一致。

## 是否存在 Flow 错误？
否。Governance Decision Flow 七步 + Evidence Flow（canonical→HMAC→prev_hash→genesis）逐符号核对通过。补充发现：异常路径的 fail-closed 分支应加入 Control Flow 图（reconciliation 已注记）。

## Bypass / Fallback 路径专项
| 路径 | 行为 | 判定 |
|---|---|---|
| scan 异常 | exit(2) 阻断（fail-closed） | ✅ 安全 |
| policy 异常 | decision=deny | ✅ 安全 |
| 日志写入异常 | pass（仅审计缺失，判定不受影响） | ⚠️ 可接受（审计缺口 vs 可用性取舍） |
| fast_screen 空/异常 | 返回 benign（RAG T1 预筛信号，非最终判定） | ⚠️ 可接受（仅影响 RAG 过滤第一阶段风险分） |
| user_id 获取失败 | "unknown"（ActivityEvent fallback） | ✅ 无害 |
| 未知 trust level | _trust_rank 返回 0 = untrusted（fail-closed） | ✅ 安全 |

## 是否发现新的 Benchmark / Regression Case？
是（3 个候选，进 Corpus 供 skill CI 参考）：
1. **异常注入回归**：mock scan 抛异常 → 断言 exit 2（验证 fail-closed 不被回归掉）。
2. **审计链空链语义**：verify_chain([]) 返回 (True, [])——空链"有效"的语义需文档化（无记录是否应报警？）。
3. **i18n LC_ALL 回归**：测试应显式 monkeypatch.delenv("LC_ALL") 后再断言 LANG 回退——作为测试隔离规范 case。

## Reconciliation 决议
- 接受 Auditor 全部补充：新增 EK-17（fail-closed 异常语义）、EK-18（_regex_guard 用户规则防护）；EK-09 补谓词取值映射（9 类 predicate 上下文查询）。
- 判定统计更新：CONFIRMED 9 + MISSING→补 2 = 11 项核心事实；NEEDS_HUMAN_REVIEW 1（i18n 测试意图）。
- 不修改原包 00-06 已写文件（保留初始考古记录）；补充内容作为 reconciliation 增量并入 Corpus 提交。
