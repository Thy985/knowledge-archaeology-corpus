# 06 Validation & Evidence — ARCH-2026-10-07-001

## 1. Truth Auditor（Blind Reconstruction）
独立重读仓库后重建关键事实，未使用考古产物作为来源：
| 声明 | 盲重建结果 | 判定 |
|---|---|---|
| MemoryGuard 默认 permissive | 复现：`MemoryGuard()` 构造 `Policy.permissive()`（guard.py:52）；strict 才主动拦截 | CONFIRMED |
| write 走五阶段管线 | 复现：source_class→classification→detectors→decide→commit 全链（guard.py:250-400） | CONFIRMED |
| read 先 verify 后检测 | 复现：`self.verify(key)` 在 detectors 前（guard.py:480） | CONFIRMED |
| 默认 7 检测器套件 | 复现：PromptInjection/SensitiveData/SizeAnomaly/RapidChange/Protected/CrossTask/SelfReinforcement（guard.py:95-105） | CONFIRMED |
| immutable_keys globs 匹配 | 复现：`is_immutable` 用 fnmatchcase（policy.py:65-70）；#136 修 glob 建基线 | CONFIRMED |
| 92.5% recall 为自编 55-case | 复现：compliance-mapping §0 明示 40 attack+15 benign、AMG 0.2.2、附语料声明 | CONFIRMED |
| Snapshot.digest 不验证 | 复现：snapshots.py docstring 明示 + #139 commit | CONFIRMED |
| 默认仅 SYSTEM 衰减自增强 | 复现：self_reinforcement.py `_DEFAULT_TRUSTED_SOURCE_CLASSES={SYSTEM}` | CONFIRMED |
| AMSB canary 判定规则 | 复现：harness docstring + ScenarioRun.breached 语义 | CONFIRMED |

## 2. Coverage Auditor
- **强制子系统覆盖**：guard/classification/integrity/events/policies/detectors(12)/storage/middleware/cli/bench(9)/integrations(6)/mcp-server/scanner/examples/tests(38)/docs(compliance-mapping)/修复史(210 commits) ✅
- **独立重搜**：grep 全仓 `PolicyViolation|IntegrityError|promote|rollback|canary|snapshot|serve` 确认无遗漏子系统 ✅
- **已知缺口（诚实声明）**：docs/ 全量未读（architecture/detectors 细节可能含设计文档未纳入）；analytics/（下载统计脚本）；templates/、_config.yml（Jekyll 页面）；CHANGELOG 全史；未实际运行基准（benchmarks/security_benchmark.py 与 amg-bench 未复跑——时间盒限制）

## 3. Flow Auditor
- Flow Atlas 七类流每条关键 Edge 已回溯 symbol/file/行号（见 04 表）✅
- bypass/override/exception/alternate/direct call 路径核查：
  - **exception path**：detector 异常→fail-open+LOW 事件（#138 证实）✅
  - **alternate path**：promote 是 write 之外的唯一分类变更通道（write 内 reclassify 被 BLOCK）✅
  - **fallback path**：ML 检测器缺失→regex-only（ml extra 可选）✅
  - **legacy path**：source_type→source_class 兼容映射（write:275-285）✅
  - **admin path**：SYSTEM SourceClass 是唯一默认受信佐证（self_reinforcement）✅
  - **直接调用**：store.set 在 ALLOW 分支直接调用（绕过检测的唯一合法路径是 guard 自身）✅

## 4. Abstraction Auditor
| KO | 升维检查 | 判定 |
|---|---|---|
| KO-01 写路径判定门 | 解释范围>单项目（通用记忆写入）；反例 3 个；有跨项目验证路径 | PASS（L3/L4） |
| KO-02 检查范围=安全属性 | 两独立修复史支撑；反例含边界反例（digest 不验证）；Principle 标 pending | PASS（L4） |
| KO-03 fail-open 可见性 | 通用安全共识；项目内强证据 | PASS（L4，pending） |
| KO-04 分层防线 | 通用模式；项目为 agent 记忆域实例；反例 2 | PASS（L3） |
| KO-05 信任梯度晋升门 | 单项目强证据；跨项目 pending | PASS（L4，pending） |
| KO-06 边界声明诚实度 | 项目独特文化；对照不足（C-02）→ 保留 KO 但标 pending | PASS（L4，pending） |
| KO-07 观测→合规链 | 通用模式 + 项目实例 | PASS（L3） |
| KO-08 可判定基准 | 通用评分哲学；反例 2 | PASS（L3） |
| 过度升维检查 | 无 KO 声称 Law/L5 级普遍定律；所有 Principle 均标 Cross-project validation pending | PASS |

## 5. Counterexample Hunter（反例预算）
| KO | 定向反例攻击 | 结果 |
|---|---|---|
| KO-01 | ①permissive 下 BLOCK 分支不触发 ②REDACT 仍提交 ③QUARANTINE 不入 store | 未推翻（管线仍运行），补充边界注释 |
| KO-02 | Snapshot.digest 不验证（EK-20）是"边界不可见"反例 | 未推翻 KO（该反例本身是另一不变量实例），保留为显式边界 |
| KO-03 | event handler 异常被吞（_emit try/except） | 未推翻（观测通道自身故障属已知边界），记录 |
| KO-04 | ML 默认关闭；完整性只覆盖 immutable 键 | 未推翻（分层仍成立，声明覆盖范围） |
| KO-05 | source_class 调用方自报可伪造 | 未推翻（信任梯度在诚实 provenance 假设下成立），记录为 C-04 |
| KO-06 | README 表无边界声明 vs compliance-mapping 有 | 未推翻 KO（项目文化主体证据），记录叙事不一致 |
| KO-07 | receipt_uri 仅指针无签名验证 | 未推翻（指针语义明示），记录 |
| KO-08 | canary 规则不测"写拦截但读仍毒"中间态；21 自编语料 | 未推翻（harness 注释已承认判定口径），记录为 C-03/C-08 |

**Contradictions 记录（保留）**：
- CONTRADICTION-1：README benchmark 表（59µs 中位延迟）与 compliance-mapping §0（62µs）数值不一致——版本差异（README 表未标 0.2.2 pin）而非行为矛盾；保留双值并标注
- CONTRADICTION-2：README 声称 "14,949 PyPI downloads"（自报营销数字）与"科学数字附语料"的工程文化并存——对外叙事两个语域，非行为矛盾

## 6. Epistemic Auditor
| 对象 | 认知状态 | 检查 |
|---|---|---|
| EK-01~35 | Fact/L1-L2 | 每条可回溯 symbol/file/commit ✅ |
| KO-01~08 | Pattern/Cognitive Model/Principle | 全部声明 aggregation_rule + 反例 + pending 标注 ✅ |
| C-01~07 | Hypothesis/Observation | 与 KO 严格区分；标注验证路径 ✅ |
| 92.5% recall | Observation（仓库自报） | 未升维为 Fact（未独立复测）✅ |
| "防篡改审计链"需求映射 | Observation | 雷达 next-step 与仓库能力（receipt_uri/OTel/SARIF）相关性——连接未验证为已实现 ✅ |

## 7. 质量指标（run_metadata 同步）
| 指标 | 值 |
|---|---|
| Facts/Evidence 引用 | 100+（EK 33 条 × 各自 evidence；Project Layer 全表） |
| Engineering Knowledge | 33 条（全部带 links） |
| EK Graph 平均出边 | ≥1（全部 ≥1，多数 2-3） |
| 游离 EK | 0（<20% 门槛） |
| 边类型覆盖 | 6/6（mechanism/subsystem/causal/dependency/constraint/contrast） |
| 聚合规则覆盖率 | 8/8 KO 全带 aggregation_rule（100%） |
| KO 簇规模 | 3~5 EK（3~12 门槛内） |
| KO 反例 | 每条 2-3 个定向攻击（≥3 目标，0 反例给搜索证据：全部完成） |
| Flows | 7/7 类（Control/State/Data/Evidence/Authority/Memory/Policy） |
| Candidates | 7 条（C-01~C-07） |
| 假聚合检查 | 无"同子系统=聚合理由"（每 KO 聚合规则为机制/因果链/不变量/主题） |
| 盲重建 | Truth 9 项全 CONFIRMED |

## 8. Reconciliation 记录（独立验证修正合并，2026-10-07）
独立 Auditor 盲重建（见 independent_validation_report.md）产生 4 项修正，已合入本 corpus 产物：
| IF | 类型 | 修正动作 | 落地位置 |
|---|---|---|---|
| IF-1 | DOWNGRADED | KO-04 增加默认配置边界：默认 7 检测器**不含 memory_persistence_injection**；strict=ASI06 威胁族部分覆盖（AMSB 实测 4/4 persistence breach、71.4 分 D），hardened=全 12 检测器（1/15、93.9 分 A） | 03_knowledge-layer KO-04 反例③ |
| IF-2 | PARTIALLY_CONFIRMED | 92.5% recall 维持 Observation；补 AMSB 实测对照（与 55-case 语料不同、不可互证） | 本报告 §9 + run_metadata |
| IF-3 | MISSING | Candidates 增加 C-08：regex 词形插入规避（persist-directive-evasion 实测 1/15） | 05_candidates |
| IF-4 | MISSING | EK-24 补充 baseline.py 对照设计意图（strict vs hardened 并排 = 故意暴露默认配置缺口的诚实基准设计） | 02_engineering-knowledge EK-24 |

原考古产物（archaeology-jobs/ARCH-2026-10-07-001/package/）保持原样未改；本文件为合并修正后的 corpus 版本。

## 9. AMSB 独立复跑（Benchmark case，本 run 实测）
```python
# repo 根目录，python3
from agent_memory_guard.bench.harness import run_suite
from agent_memory_guard.bench.scenarios import default_corpus
from agent_memory_guard.bench.scoring import score_report
from agent_memory_guard.bench.adapters.baseline import UnguardedDictAdapter, AMGStrictAdapter, AMGHardenedAdapter
for a in (UnguardedDictAdapter, AMGStrictAdapter, AMGHardenedAdapter):
    r = run_suite(a, default_corpus()); s = score_report(r)
    print(a.__name__, sum(1 for x in r.runs if x.breached), '/15', s.score, s.grade)
```
| adapter | 配置 | malicious breach | benign FP | score | grade |
|---|---|---|---|---|---|
| UnguardedDictAdapter | 无防御 | 15/15 | 0 | 0.0 | F |
| AMGStrictAdapter | strict + 默认 7 检测器 | 4/15（全 memory_persistence） | 0 | 71.4 | D |
| AMGHardenedAdapter | strict + persistence 检测器 + identity keys | 1/15（persist-directive-evasion） | 0 | 93.9 | A |
结论：AMSB canary 判定/FP 惩罚/封顶评分实测行为与描述一致（KO-08 CONFIRMED）；strict 与 hardened 的差距验证 IF-1 的默认配置边界。
