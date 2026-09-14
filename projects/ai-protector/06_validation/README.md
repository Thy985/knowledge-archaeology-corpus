# 06 — Validation & Evidence

> 阶段 4 内部验证（Truth/Coverage/Flow/Abstraction/Counterexample/Epistemic）。阶段 5 独立 Auditor 盲重建见 independent_validation_report.md。

## 1. Source Truth（事实真值）

| Claim | 判定 | 证据 |
|---|---|---|
| LangGraph 9 节点管线 | ✓ CONFIRMED | graph.py:22（9 个 add_node） |
| 三态路由 BLOCK/MODIFY/ALLOW | ✓ CONFIRMED | graph.py:18 route_after_decision + test_graph 路由用例通过 |
| 加权风险分阈值可配 | ✓ CONFIRMED | decision.py calculate_risk_score（thresholds.get 默认值） |
| NeMo 从硬阻断改软贡献 | ✓ CONFIRMED | decision.py:112-118 注释（明确修复史） |
| A2 反混淆 raw 恒首位 + variants | ✓ CONFIRMED | deobfuscate.py:200-260 |
| variants-first 降检测 ~13pp | ✓ CONFIRMED（项目实验记录） | deobfuscate.py:252-255 注释（S3 实现内嵌实验） |
| 输出过滤三防线 | ✓ CONFIRMED | output_filter.py（PII/secrets/system-leak） |
| pre-tool 5 项 / post-tool 3 项 | ✓ CONFIRMED | pre_tool_gate.py / post_tool_gate.py 函数清单 |
| RBAC 模型（sensitivity/confirmation/inherits） | ✓ CONFIRMED | rbac/models.py |
| ISS-003/004 幻影功能 | ✓ CONFIRMED | issues.md:37-67（未修复状态） |
| 基准数字 99%/91%/48ms | ⚠️ 项目自述（S7），本地未复现 benchmark | README/BENCHMARKS.md——如实标注 |
| 1900+ 测试 / 83% 覆盖率 | ⚠️ 项目自述（S7） | 本地实测 668 passed（DB 依赖不可复现） |

## 2. Coverage（覆盖）

- 检测层：7 层全读（rules/intent/scanners 5 扫描器/decision）✓
- 输出侧：output_filter ✓；agent 层：pre/post gate + RBAC + graph ✓
- 失败史：issues.md 13 条全读 + CHANGELOG 0.2.0-0.2.8 ✓
- Benchmark 基建：BENCHMARKS.md 431 行全读 + red_team 结构 ✓
- 治理：SECURITY.md + THREAT_MODEL.md + workflows ✓
- **未覆盖**：frontend（Nuxt，0 py）；red_team 内部实现细节（run engine/SSE/SecretStore 未逐行）；infra/scripts——已在 01 层标注。

## 3. Flow（流程真值）

- 7 类流全部回溯文件:行（04_flow-atlas 自检）✓
- 关键风险点：MODIFY 路径实存（graph.py add_edge transform→llm_call）✓；pre_llm_only 实存（runner.py:143）✓；BLOCK 短路不经过 llm_call ✓

## 4. Abstraction（升维纪律）

- 10 KO 全部带 aggregation_rule（R1-R4）+ 簇内 EK + 边 ✓
- 跨项目陈述数对项目数：KO-01（3 项目）、KO-02（2 项目，明确标注）、KO-03（2 项目）、KO-09（3 项目编排对比）✓
- KO-06/KO-10 标 `Cross-project validation pending` ✓
- 未因"结论漂亮"升层：MODIFY 三态仅 L3（证据 2 项目）✓

## 5. Counterexample（反例压力）

| 升维候选 | 定向反例 | 结果 |
|---|---|---|
| "确定性判定可审计"（KO-01） | ISS-003/004 幻影功能（UI 声明 ≠ 实现）——可审计性受 UI 误导威胁 | 未推翻 KO-01（判定管线本身确定），但收紧：可审计仅指判定层，UI 层存在幻影 |
| "规则层容错"（KO-03） | ISS-005 keyword-only intent 被改写绕过——**检测盲区不崩但会漏** | 未推翻（容错≠不漏），明确边界：容错是可用性保证不是覆盖保证 |
| "No LLM in the loop" | config.py 本地 ML 模型（granite-guardian 2B 等）——README 措辞有歧义 | 未推翻字面（无外部 LLM API），标 C-03 语义边界待澄清 |
| "provable"（KO-06） | 基准数字为项目自述（S7）非独立复现 | 未推翻设计意图，但证明强度标注为 S7；C-07 记录 668 vs 1900 差距 |

## 6. Epistemic（认知状态标注）

- Hypothesis 不冒充 Fact：C-01~C-09 全标 Hypothesis/Tentative ✓
- Pattern 不冒充 Principle：KO-01~09 为 L3/L4，跨项目 pending 明确 ✓
- 项目自述 vs 本地实测分离：基准/覆盖率标 S7，测试标 S4 实测 ✓
- 修复史标注证据级：ISS-001/002/024（项目记录已修，S7）vs 代码注释（S3）分开 ✓

## 7. 质量指标

- 本地复现测试：**668 passed**（proxy 纯逻辑 203 + agent-demo 465；--noconftest 绕过 DB fixture）
- 环境受限：proxy DB 依赖测试（24）+ scan_via_proxy（23）+ 358 场景（353 errors）——无 Postgres/Redis/服务，非产品失败
- EK：36 条，links 全覆盖（平均出边 ≥1）✓
- KO：10 个，聚合规则 100% ✓
- Candidates：9 条 ✓
- 已知未验证：frontend 逐组件、red_team 内部逐行、358 场景本地全量

---

## Reconciliation 记录（阶段 6）

- **输入**：阶段 4 Validation + 阶段 5 Independent Validation Report（判定：8 CONFIRMED / 2 PARTIALLY / 3 MISSING / 2 NEEDS_HUMAN_REVIEW / 0 DOWNGRADED / 0 OVER_GENERALIZED / 0 CONTRADICTED）。
- **采纳修正**：
  1. 02 层新增 EK-36/37/38（direct bypass / streaming 无输出过滤 / 管理面无认证）——补覆盖 Auditor 攻击面。
  2. KO-08 措辞收紧："输出侧最完整" → "输出侧三线为 AI Protector 独有；与 Aigis exfil 对照待定"。
  3. 04 层 Flow Atlas 补充 bypass flow（direct 旁路 + streaming 输出旁路）。
- **未采纳**：无（Auditor 判定全部接受）。
- **保留**：基准数字 S7 自述标注（不改为实测）；C-07（1900+ vs 668）保持 Candidate。
- **结果**：整体 CONDITIONAL PASS——核心机制全代码级确认；绕过/管理面覆盖缺口已补录；生产部署安全性 2 项 NEEDS_HUMAN_REVIEW（enable_direct_endpoint 默认 True + 管理面无认证）。
