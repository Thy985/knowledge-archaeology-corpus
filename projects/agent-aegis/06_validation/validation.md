# 06 Validation & Evidence — agent-aegis（ARCH-2026-10-22-001）

## Truth Audit（真值）
- 18 条 EK 全部回溯仓库实盘（HEAD 134580e，文件:行锚）；无"文档当实现"（policy.example.yaml 为配置样例非实现；AGEF/AGP 标注 Draft 自述）
- README 徽章（6400+ tests / 92% coverage）为外部 CI 声明——标注为项目自述，未独立复跑（覆盖审计缺口）

## Coverage Audit（覆盖）
- 已覆盖：接入面（G01/G02/G18）/ 判定链（G03/G07/G08）/ 策略面（G04/G05/G06/G10）/ 审计（G11）/ 身份（G12/G15）/ MCP 攻击面（G13/G14）/ 标准（G17）/ 降级（G16）
- 未深读（声明缺口）：guardrails 各具体 guardrail 实现（PII/injection 规则内容）、hazard_classifier 类别表、policy_git 热更新、cost_policy/budget 细节、rbac/sandbox_policy 全貌、selection_audit 内部、mcp 各检测器默认配置、packages/*-aegis 独立包

## Flow Audit（流真实性）
- 七类流全部从代码导出；关键 Edge 可回溯：
  - _is_cacheable 分支（policy.py:213-228）
  - monotone 平凡真（len<2 → True，action_claim.py:109-121）
  - init 失败降级（warning 不阻断，init.py:195-225）
  - _patch_openai/_patch_anthropic 可逆（init.py:328-389）

## Abstraction Audit（升维诚实性）
- L4 3 个（KO-G01/G03/G05）+ L3 2 个（KO-G02/G04）——无 L5（不硬凑）
- 所有 L4 满足"解释范围扩大"论证 + 五条件；KO-G03 明确与 Archon opt-in gate 对照为"两种接入哲学"而非优劣结论
- 无单案例→Pattern 跳跃（C-G05/G06 保持 Cross-project pending）

## Counterexample Hunt（反例）
- KO-G01 反例：_assess=False 无独立评估路径（诚实降级，声明字段直接进 policy）
- KO-G02 反例：merge defaults 取 base——环境无法覆盖默认风险级别（语义边界）
- KO-G04 反例：shadow 检测依赖 trusted_servers 白名单（未配置时能力降级，需验证默认）
- KO-G05 反例：OpenSSF 徽章是外部程序但非 AGEF/AGP 生态采纳（反例确认不变量边界）

## Epistemic Audit（认知状态）
- C-G01~G07 全标 Hypothesis/Observation；无"结论漂亮就升层"
- AGEF/AGP 标注"自述 Draft"（非 Fact 生态）；README 徽章标注"项目自述"
- 文档-实现对照：AGP Level1 conformance 以真实引擎验证（test_agp_level1.py）——文档（protocol.md）与实现（policy/action 模型）一致；AGEF schema 由 test_agef_schema 钉住

## 质量指标（初版）
- EK 18 条（G01~G18）/ KO 5 条（R1×2 R2×1 R3×1 R4×1）/ Candidates 7 / Flow 7 类 / evidence 引用 ~55 处
- EK links 覆盖率 100%；平均出边 ~1.8；游离 0%；聚合覆盖率 100%
- 反例预算：5 个 KO 各 ≥1 定向反例（3 个内置 + 2 个 C 候选确认）

## Contradictions 保留
- 无实现内部矛盾确认；已知**文档-实现缺口**：README 声称 "12 frameworks" 但 instrument 列出 10 patch_* + 2 raw API 客户端（对应 12）——计数口径需 Auditor 核实；AGEF 声称"interoperable across tools"为愿景（自述）非已证事实
