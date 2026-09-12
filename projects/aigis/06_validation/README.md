# 06 — Validation & Evidence

## 验证方式
- 测试实测：`python3 -m pytest tests/ -q --tb=no` → **1 failed, 1711 passed, 1 warning in 67.05s**。
- 失败分析（test_i18n.py::TestDetectLang::test_unknown_env_falls_through）：测试设置 AIGIS_LANG=fr + LANG=ko_KR.UTF-8 期望 "ko"；实测 "en"；根因 = 沙箱全局 `LC_ALL=en_US.UTF-8`，detect_lang 按 POSIX 标准 LC_ALL>LANG 优先（i18n.py:154-166），测试未 mock LC_ALL → **测试环境耦合缺口，非产品缺陷**（spy 实测：normalize('en_US.UTF-8')->'en'，逻辑与源码一致）。同类问题：agentevals 轮的 pytest-asyncio 缺口——测试环境假设族。

## Truth Auditor（源真实）
| 声明 | 判定 | 证据 |
|---|---|---|
| 零核心依赖 | ✅ CONFIRMED | pyproject `dependencies = []` |
| HMAC + hash chain 防篡改 | ✅ CONFIRMED | audit/signed_log.py + chain.py + test_tampered_* 族 |
| L1-L7 管线 | ✅ CONFIRMED（架构图） | ARCHITECTURE Detection Pipeline + filters/safety/spec_lang 代码 |
| 93.5% / 0.0% FP | ⚠️ PARTIALLY（项目自述） | README benchmark 表（S7，未实测复跑） |
| 53 项韩国合规映射 | ⚠️ PARTIALLY | pyproject description（S7）；compliance_kr.py 存在（S3） |
| 8 天 21 patch ≈60 detectors | ⚠️ PARTIALLY（自述） | README 段落（S7）；未复验 commit 史（浅克隆 depth=1） |

## Coverage Auditor（覆盖）
- 已覆盖：检测管线（L1-L7）、MCP 安全、审计防篡改、PolicyDSL、Incident、日志架构、治理文档、合规模板、多 Agent/跨会话/记忆、对抗循环、i18n。
- 未深读（不影响核心结论）：redteam.py 内部实现、middleware/ 各适配器细节、memory/ 检测器细节、backend/frontend/sdk/vscode-extension（非核心包）。
- 重要发现无遗漏：测试失败已单独归因；fork/上游差异已进 C-03。

## Flow Auditor（流真实性）
- Governance Decision Flow 七步全部对齐代码（适配器→ActivityEvent→管线→policy→日志→exit code）。
- Evidence Flow 链（canonical→HMAC→prev_hash→genesis→verify_chain）逐行对齐 audit/ 源码 + 测试。
- 未发现"想象出来的流"。

## Abstraction Auditor（升维检查）
- 4 个 KO 全部 L3/L4（无 L5）——单项目证据不足不升方法论；均标 `Cross-project validation pending`。
- 无"单案例→Pattern"违规（KO 均有 ≥2 EK 且跨子系统）。

## Counterexample Hunter（反例）
- KO-01（确定性优于概率）：反例 = 确定性规则对未知攻击覆盖有限（10 miss 自证）——已进 C-04 张力。
- KO-02（正交分层）：反例 = L4 taint 与 L5 sandbox 有功能重叠（都涉及"阻止危险执行"）——分层正交性部分成立，非全正交；已标注。
- KO-04（零依赖）：反例 = 可选依赖（cryptography/pyyaml/fastapi）在 enterprise 模式成为依赖面——零依赖仅核心路径成立，已标注。

## Epistemic Auditor（认知状态）
- Fact/Observation/Hypothesis/Pattern 区分合规：EK-01..16 为 Fact（S3/S4）；KO-01..04 为 Pattern/Model（S3+S4，跨项目 pending）；C-01..04 为 Hypothesis/Tentative。
- **Hypothesis 未冒充 Fact**：93.5% 等基准明确标"项目自述"；fork 能力差标"未量化"。
- **Cross-project Candidate 未写成已验证 Principle**：C-01 明确为交叉项目假说。

## 反例预算
- 3 个 L3+ KO 各 ≥2 定向反例：KO-01（10 miss）、KO-02（L4/L5 重叠）、KO-04（可选依赖）→ 反例预算消耗 6/9。

## 质量指标
| 指标 | 值 |
|---|---|
| EK 数量（links 覆盖率） | 16（100% 有 links；平均出边 >1；孤立 0） |
| KO 数量（聚合规则覆盖率） | 4（100% R1-R4） |
| Candidates | 4（2 Hypothesis / 1 Tentative / 1 L3 候选） |
| 测试 | 1711 passed / 1 failed（67s） |
| 失败归因 | 测试环境耦合（LC_ALL 未 mock），非产品缺陷 |
| 反例预算 | 6/9 消耗 |

---
## Reconciliation（Auditor 补充并入，2026-09-13）

| 项 | 决议 |
|---|---|
| EK-17（fail-closed 异常语义） | ✅ 并入 02（claude_code.py:93-111：scan/policy 异常→阻断；日志异常→pass） |
| EK-18（_regex_guard 用户规则防护） | ✅ 并入 02（防 re.error / 静默禁用 / ReDoS；None=hard failure） |
| EK-09 补充 | ✅ 谓词取值映射（9 类 predicate 上下文查询：resource/target/risk/taint/session_age/action_count/tool_name/contains） |
| 4.1 Control Flow 补充 | ✅ fail-closed 异常分支已注记（scan/policy 异常 → exit 2 阻断） |
| Benchmark/Regression 候选 | 3 个（异常注入回归 / 空链语义 / LC_ALL 测试隔离规范），见 independent_validation_report.md |

判定统计（含 Reconciliation）：CONFIRMED 11 / PARTIALLY_CONFIRMED 2 / MISSING→补 2 / CONTRADICTED 0 / OVER_GENERALIZED 0 / NEEDS_HUMAN_REVIEW 1（i18n 测试意图：LANG 应胜还是 LC_ALL 应胜，需维护者确认）。
