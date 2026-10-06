# Independent Validation Report — ARCH-2026-10-07-001（OWASP Agent Memory Guard）

> Auditor 角色：不把考古产物当事实来源，独立重读仓库建立 Independent Findings 后与 package 对比。**未修改任何原考古产物。** 本报告独立产出，供 Reconciliation 合并。

## 0. 方法
- 独立抽查 10 项关键声明（A1-A10）：默认策略/read 顺序/detector 异常/快照 digest/默认检测器套件/regex 数量/leakage 数量/self-reinf 参数/promote 门/检测器文件数
- 运行 AMSB（仓库自带 benchmark）对 3 个官方 adapter 实测，验证判定规则与评分模型
- 与 package（00-06 + run_metadata）逐项对比，重点攻击：单案例→Pattern、Pattern→L4、Flow Edge 真实性、Epistemic 混淆

## 1. 判定统计
| 判定 | 数量 | 条目 |
|---|---|---|
| CONFIRMED | 10 | A1-A10 全部 |
| PARTIALLY_CONFIRMED | 1 | IF-2（92.5% recall 语境） |
| DOWNGRADED | 1 | IF-1（KO-04 分层防线 scope） |
| OVER_GENERALIZED | 0 | — |
| MISSING | 2 | IF-3（regex 规避变体证据）、IF-4（诚实基准对比设计） |
| CONTRADICTED | 0 | — |
| NEEDS_HUMAN_REVIEW | 0 | — |

## 2. 独立抽查 A1-A10（全部 CONFIRMED）
| # | 声明 | 独立验证 | 结果 |
|---|---|---|---|
| A1 | 无参 MemoryGuard 默认 permissive | `guard.py:80 policy or Policy.permissive()` | CONFIRMED |
| A2 | read 先 verify 再检测 | `guard.py:445 read → 451 verify(key)` 在检测器前 | CONFIRMED |
| A3 | detector 异常 fail-open 且发事件 | `guard.py:621 detectors must never break + 636 detector_error=True` | CONFIRMED |
| A4 | Snapshot.digest 只记录不验证 | `snapshots.py:21 forensics only` | CONFIRMED |
| A5 | 默认 7 检测器套件 | `guard.py:99-107` 列表实测 7 个 | CONFIRMED |
| A6 | 11 条注入 regex | `injection.py` 计数=11 | CONFIRMED |
| A7 | 13 类泄漏 pattern | `leakage.py` 计数=13 | CONFIRMED |
| A8 | self-reinf 冷却 60s/3 次 + 默认仅 SYSTEM 受信 | `self_reinforcement.py:74-75 + :32/61` | CONFIRMED |
| A9 | 12 检测器（13 文件含 base 协议） | `detectors/*.py`=13，base 为 Protocol | CONFIRMED |
| A10 | promote verified 门 | `guard.py:210 requires_verification and not verified → raise` | CONFIRMED |

## 3. 3 个成功（package 高置信条目，实测支持）
1. **写路径判定门**（KO-01）：实测 guard.write 走 source_class→classification→detectors→decide→commit 全链；PolicyViolation 在 BLOCK 分支抛（adapter remember 捕获 → BLOCKED outcome 实测生效）
2. **AMSB 判定/评分模型**（KO-08）：实测 unguarded 21 scenarios → 15/15 breach、0 FP、score 0.0、grade F——canary 判定与 FP 惩罚行为与 package 描述一致
3. **完整性先行的读路径**（KO-01/Flow）：guard.py:451 verify 在检测器前——Flow Atlas Control.read 边 `verify → detectors → decide` 真实存在

## 4. 3 个错误/修正（auditor 对 package 的挑战）
### IF-1 DOWNGRADED（KO-04 scope 收紧——需 Reconciliation）
- **发现**：`baseline.py:11-14` 明示 `AMGStrictAdapter`（官方推荐 enforcing 配置）**故意不加载 opt-in 的 persistence detector**；实测默认套件=7 个（A5），**不含 memory_persistence_injection**
- **实测**：AMSB strict=**4/15 breach（全部 4 个 memory_persistence scenario）**、score 71.4、grade D；hardened=**1/15 breach**（persist-directive-evasion）、score 93.9、grade A
- **对 package 的影响**：KO-04"各层失败由下一层兜底"的分层防线表述在**默认配置语境下过度概括**——ASI06 核心威胁（延迟生效持久化指令）在 out-of-the-box enforcing 配置下不设防（仓库自己称"standard install stops"）。EK 事实均准确（12 检测器 vs 默认 7 套件分开陈述），但 KO-04 需补充默认配置边界：**默认防线=7 检测器威胁族；persistence 注入需显式 opt-in（hardened）**

### IF-2 PARTIALLY_CONFIRMED（92.5% recall 的语境）
- **发现**：README 表（92.5% recall / 59µs）与 compliance-mapping §0（62µs，pinned 0.2.2）两套数字；package 已把 92.5% 标为 Observation
- **实测**：AMSB 21 语料下 strict=71.4/D、hardened=93.9/A——**与 55-case 92.5% 不可互证**（语料与判定口径不同：AMSB canary 存活判定 vs 检出率）
- **结论**：维持 Observation；补充 AMSB 实测对照值作为独立证据（见 §6）

### IF-3 MISSING（regex 规避变体的对抗证据）
- **发现**：hardened 下唯一 breach 是 `persist-directive-evasion`（"Directive promotion with an adjective inserted to evade fixed patterns"）
- **package 缺口**：C-03/C-08 只提"未知攻击族"，未提**已知族内 regex 规避变体**的实测反例——regex 检测对抗边界的直接证据在仓库内（scenario 定义 + 实测 1/15）
- **建议**：Candidates 增加"regex 检测可被词形插入规避（persist-directive-evasion 实测）"

## 5. 遗漏（auditor 重搜发现）
### IF-4 MISSING：诚实基准对比设计本身是知识资产
- `baseline.py:1-22` 模块 docstring 完整解释"并排报告 strict/hardened 是 'how good is AMG' 的诚实版本；gap 是 persistence detector 存在的原因"——**基准设计决策**（官方不隐藏默认配置的缺口，反而把它做成对照实验）未进 package 01/02
- 建议：Reconciliation 在 EK-24 补充此设计意图（strict vs hardened 对照 = 故意暴露默认配置边界）

## 6. Benchmark case（AMSB 实测，可复现）
```bash
cd repo && python3 -c "import sys; sys.path.insert(0,'src');
from agent_memory_guard.bench.harness import run_suite;
from agent_memory_guard.bench.scenarios import default_corpus;
from agent_memory_guard.bench.scoring import score_report;
from agent_memory_guard.bench.adapters.baseline import UnguardedDictAdapter, AMGStrictAdapter, AMGHardenedAdapter;
for a in (UnguardedDictAdapter, AMGStrictAdapter, AMGHardenedAdapter):
    r = run_suite(a, default_corpus()); s = score_report(r)
    print(a.__name__, 'breach=', sum(1 for x in r.runs if x.breached), '/15', 'fp=', sum(1 for x in r.runs if x.false_positive), 'score=', round(s.score,2), s.grade)"
```
| adapter | 配置 | malicious breach | benign FP | score | grade |
|---|---|---|---|---|---|
| UnguardedDictAdapter | 无防御 | 15/15 | 0 | 0.0 | F |
| AMGStrictAdapter | strict + 默认 7 检测器 | 4/15（全 memory_persistence） | 0 | 71.4 | D |
| AMGHardenedAdapter | strict + persistence 检测器 + identity keys | 1/15（regex 规避变体） | 0 | 93.9 | A |

结论：AMSB 判定/评分/FP 惩罚模型实测行为与 package 描述一致（KO-08 CONFIRMED）；同时产出默认配置威胁族缺口证据（IF-1/IF-3）。

## 7. Reconciliation 请求（供阶段⑥合并，禁改本报告与 package 原文）
1. KO-04 增加默认配置边界声明（默认 7 检测器不含 persistence；strict=ASI06 威胁族部分覆盖，hardened=全 12 检测器）
2. 06 Validation 增加 AMSB 三 adapter 实测对照（本报告 §6）
3. Candidates 增加 C-08：regex 检测词形插入规避（persist-directive-evasion 实测 1/15）
4. EK-24 补充 baseline.py 对照设计意图（IF-4）
5. run_metadata quality_metrics 补充 independent_validation 字段
