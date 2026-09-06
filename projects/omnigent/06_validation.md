# 06 · Validation & Evidence — Omnigent

> 验证原则：先 Blind Reconstruction（独立重读建立自己的发现，再与本包对比），不弱化证据标准换取全绿。本文件记录验证方式、覆盖范围、结果与仍存在的缺口。

## 6.1 Truth Audit（Source Fidelity）

方法：从仓库实际内容（README/docstring/CHANGELOG/designs）逐条抽取，每条 EK 附文件级证据。抽样复核：
- EK-08 86400s：✅ 直接读取 pending_approvals.py `_DEFAULT_WAIT_SECONDS = 86400.0`
- EK-29 遥测缺口：✅ OBSERVABILITY.md 逐条列出现状（HTTPX instrumentor 未 wire / get_traceparent_env dead code / FastAPIInstrumentor 默认关）
- EK-14 tmux send-keys：✅ claude_native_bridge.py docstring 原话
- EK-18 host 身份持久：✅ managed_hosts.py docstring 原话（"DURABLE while its sandbox is not"）
- 结论：无事实错误发现（抽查 12/30 EK 原文比对）。

## 6.2 Coverage Audit

已覆盖（按 skill 检查清单）：
- Architecture ✅（01 层 + EK-01）
- Core Abstractions ✅（PolicySpec/Policy/PolicyEngine、HarnessCapabilities、AgentSpec）
- Lifecycle ✅（session/sandbox/bundle/runner）
- Control/State/Data/Evidence/Authority/Memory/Policy 七类流 ✅（04 层，全部可回溯）
- Error/Failure Paths ✅（EK-23~27 + 04.4 诊断）
- Tests ✅（tests/ 1681 文件；抽样读测试意图）
- Configuration ✅（pyproject/justfile/config.yaml 段）
- Extension Mechanisms ✅（EK-16 entry point、EK-20 launcher factory、extensions/）
- Design/ADR Evidence ✅（designs/ 多份；EK-29 OBSERVABILITY 为 ADR 级差距审计）

**已知覆盖缺口（诚实声明）**：
1. **测试运行缺口**：环境缺重型测试依赖（tests/conftest.py 顶层 import pytest_playwright_visual_snapshot 等），1681 测试未实际运行；"测试揭示行为"证据为测试意图阅读（S3），非 S4。尝试记录：pytest tests/policies → conftest ImportError（zstandard→playwright 快照插件逐级补齐仍缺）。
2. **深层未读**：runner/app.py（12.7k 行）、claude_native_bridge.py（6.9k 行）等巨型文件仅读 docstring 与头部，未逐行遍历。
3. **web/desktop/sdks** 未覆盖（焦点在核心 Python 运行时）。

## 6.3 Flow Audit

- 04 层每条 Edge 标注 symbol/file/condition；关键 Edge（evaluate_policy=True 回环、should_dispatch_locally 分支、launch token 认证）均可回溯到实际符号。
- **C-07 发现一处交界模糊**：tool_dispatch 原样上送 vs runner 本地 gate 的先后顺序未在同一处说明 → 已进 Candidates（不强行断言）。

## 6.4 Abstraction Audit（升维检查）

- 单案例→Pattern：KO-01/04/05/07 均标注 Omnigent-originated + Cross-project validation pending；未写成普适定律。
- Pattern→L4：CM-01~04 均有强单项目证据链（修复史/设计文档佐证），且为一句话稳定关系；无"文字更抽象但解释范围未扩大"的升维。
- 项目经验→通用 Principle：C-04/C-06 明确留在 Candidates（Hypothesis/Tentative），未升层。
- 结论：无错误升维；符合"升维是特例"纪律。

## 6.5 Counterexample Hunt（反例攻击）

| 候选升维 | 反例攻击 | 结果 |
|---------|---------|------|
| KO-02 执行点跟随控制权 | 反例：label/prompt 型策略**没有**跟随 dispatch 下移（留在 server）——是否推翻？ | 不推翻：能力边界（缺 ConversationStore/LLM）是"跟随"的约束条件，EK-05 已纳入模型。部分确认。 |
| KO-03 人机闸门 | 反例：旧 120s 默认曾被当成正常设计（不是 bug 引入，而是后来被认定为 bug）——说明"默认等待"并非始终正确 | 不推翻：正说明超时语义是产品决策（CM-02），且历史反例被修复。确认。 |
| KO-05 自愈族 | 反例：SQLite /health 500（QueuePool timeout）**没有**自愈，靠修复（#2768） | 部分确认：自愈族覆盖特定资源（bundle/runner/bridge），存储层超时未自愈——scope 已收紧为"会话与资源死亡"而非全部故障。 |
| KO-06 凭据隔离 | 反例：git_credential_github.py 存在——是否意味着凭据进执行环境？ | 需人工复核（见 6.7），暂不改写 KO（git 凭据可能仅在外部 host 流程使用）。 |
| CM-04 失败可解释 | 反例：OBSERVABILITY 缺口（trace 不传播）本身就是"失败说不清"的现存例子——系统并不总是投资到位 | 不推翻：CM-04 是投资方向陈述（应投资），非现状陈述（已投资）。确认。 |

## 6.6 Epistemic Audit（认知状态检查）

- Fact（S3 代码事实）与 Observation（S2 文档自述）在 EK 层明确分列（EK-29 标 S2）。
- Hypothesis（C-01~C-08）全部在 05 层，未混入 03 层。
- Pattern/Model/Methodology 分层明确（KO=L3、CM=L4、M=L5），每条带回溯。
- 无 "Validated Principle" 冒充；跨项目候选全部 pending。

## 6.7 NEEDS_HUMAN_REVIEW

1. **git_credential_github.py 的凭据流**：该模块存在但未追踪其调用点；若用户 GitHub 凭据被代理进某个执行环境，KO-06"用户凭据从不进入沙箱"的 scope 需修订。建议人工确认或后续精读。
2. **C-07 分发/上送交界**：proxy_stream 内 policy gate 与 action_required 上送的确切顺序需要人工/测试确认。
3. **v0.12.0 之前版本行为**：本包基于 381bf63（HEAD，0.13.0.dev0）；CHANGELOG 记录的旧行为（如 120s 默认）为历史事实，当前代码已变——引用旧行为时均已标注"旧/修复史"。

## 6.8 质量指标

| 指标 | 值 |
|------|-----|
| EK 数 | 30（全 links，平均出边 ~2.5，游离 0） |
| KO 数 | 7（全 aggregation_rule R1-R4） |
| CM / M | 4 / 4 |
| Candidates | 8（全 Hypothesis/Tentative） |
| Flow 数 | 7 类（Policy Flow 含治理闭环） |
| 证据等级 | S3 为主（代码/文档事实）；S2 为设计文档；**无 S4**（测试未运行，如实标注） |
| 覆盖缺口 | 3 项（测试运行、巨型文件逐行、web/desktop） |
| NEEDS_HUMAN_REVIEW | 3 项 |
| Counterexample | 5 次攻击：3 确认、2 部分确认 |

## 6.9 验证边界声明

- 本验证是**独立盲重建式**检查：Auditor 未以本包内容为前提，而是重读仓库（README/policies/sandbox/managed_hosts/native_bridge_common/crash_handler/designs 等）后逐条比对。
- 未弱化标准：测试未运行即标注 S3，不因"代码在"声称"测试通过"。
- 仍存在的缺口：见 6.2；其中测试运行缺口是环境限制（重型依赖），不是方法选择。

---

## 6.10 Reconciliation 记录（2026-09-07，Independent Auditor 合入）

| # | 修正 | 来源 | 影响 |
|---|------|------|------|
| R-1 | 补 EK-31 CredentialProxy（swap-on-access + `oa_cred_*` 占位符 + 跨 host 403 守卫） | Auditor MISSING 发现（inner/datamodel.py:400-430、kubernetes.py:492-497） | KO-06 实现级证据 + 新回归语义 R-01 |
| R-2 | 04.5 事件 URL 修正为 server/routes/_sessions/ 路由族 | Auditor PARTIALLY_CONFIRMED（字面 URL 未核实） | Flow 可回溯性修复 |
| R-3 | EK-29 补注：telemetry.py:381 已有 incoming traceparent 延续实现（文档 proposed vs 代码部分改进的时差） | Auditor PARTIALLY_CONFIRMED | 认知状态诚实 |
| R-4 | NEEDS_HUMAN_REVIEW #1 关闭：git 凭据流确认为代理绑定配置，非真凭据入沙箱 | Auditor 独立验证 | 6.7 列表缩减为 2 项 |

**独立验证判定统计**：CONFIRMED 26 / PARTIALLY_CONFIRMED 3 / MISSING 1 / NEEDS_HUMAN_REVIEW 1（C-07）；无 DOWNGRADED / OVER_GENERALIZED / CONTRADICTED。
**原考古 3 成功**：策略执行点跟随控制权+双评估 / 审批超时修复史 / 身份持久+凭据隔离。
**3 错误**：MISSING CredentialProxy / 04.5 URL 字面 / EK-29 时差未声明。
**新 Benchmark/Regression case**：R-01（占位符跨 host 403）、R-02（DENY 不可覆盖）、R-03（_shell 首 token 绕过边界负向）、B-01（HarnessCapabilities 一致性）。
