# Independent Validation Report — ARCH-2026-10-22-001（agent-aegis）

> Auditor 盲重建：不引用考古结果，独立重读仓库建立 Findings 后对比。禁止修改原考古产物（修正经 Reconciliation 落地）。

## 判定统计（12 项独立抽查）

| 判定 | 数量 | 涉及 |
|---|---|---|
| CONFIRMED | 11 | EK-G01/G02/G03/G04/G05/G06/G07/G08/G11/G12/G13/G14/G15/G16/G17（抽查覆盖） |
| PARTIALLY_CONFIRMED | 1 | EK-G16（空配置/默认 guardrails 语义未逐行核） |
| MISSING | 1 | AEGIS_* 环境变量配置面（EK-G02 未完整覆盖 env 配置族） |
| DOWNGRADED | 0 | — |
| OVER_GENERALIZED | 0 | — |
| CONTRADICTED | 0 | — |
| NEEDS_HUMAN_REVIEW | 0 | — |

## 独立抽查证据（12 项）

| # | 抽查事实 | 独立证据 | 判定 |
|---|---|---|---|
| 1 | 惰性 import | __init__.py:26 `_LAZY_IMPORTS` + __getattr__（line 236） | CONFIRMED |
| 2 | AEGIS_INSTRUMENT 环境变量 | instrument/__init__.py:349-365 `_maybe_auto_instrument`（1/true/yes/on + AEGIS_ON_BLOCK/AEGIS_AUDIT/AEGIS_GUARDRAILS） | CONFIRMED（S-1） |
| 3 | 单调约束 | action_claim.py:109-121（len<2 平凡真 + 非递增） | CONFIRMED |
| 4 | _is_cacheable 防遮蔽 | policy.py:213-228（conditions + regex 匹配 → 不缓存） | CONFIRMED |
| 5 | claim_policy 四步 + assess 降级 | claim_policy.py:152-170（_assess=False 时声明字段直接进 policy） | CONFIRMED |
| 6 | AGP fail-safe | tests/conformance/test_agp_level1.py:6,42,59,99（Approval.BLOCK 默认 + 断言） | CONFIRMED |
| 7 | AGEF/AGP Draft 自述 | specs/agef/v1/README.md:3 + specs/agp/v1/protocol.md:3（Status: Draft） | CONFIRMED |
| 8 | MCP shadow/escalation | core/mcp_shadow.py + core/mcp_escalation.py 存在 | CONFIRMED |
| 9 | ZeroTrustAgent 组件 | zero_trust_agent.py:66-155（5 组件） | CONFIRMED |
| 10 | merge defaults 取 base | policy.py:231-246（"Defaults come from this policy (the base)"） | CONFIRMED |
| 11 | autopolicy 生成器 + 校验 | autopolicy_llm.py:117（_validate_policy_dict）+ 3 处 generate（181/257/339） | CONFIRMED（S-3） |
| 12 | 12 frameworks 声明 | 10 个子模块 patch_*（_crewai/_dspy/_google_adk/_google_genai/_instructor/_langchain/_litellm/_llamaindex/_openai_agents/_pydantic_ai）+ integrations/patch_anthropic.py + patch_openai.py | CONFIRMED（S-2，初判异常后核实） |

## 3 个成功（Auditor 支持性强化）

### S-1 · AEGIS_* 环境变量配置族（CONFIRMED 强化）
- **发现**: `_maybe_auto_instrument`：`AEGIS_INSTRUMENT`（1/true/yes/on）+ `AEGIS_ON_BLOCK`（默认 **"raise"**）+ `AEGIS_AUDIT` + `AEGIS_GUARDRAILS`（默认 "default" 暗示内置默认 guardrail 集）
- **影响**: EK-G02 补 env 配置入口细节（on_block 默认 raise——fail-closed 语义，非 warn）

### S-2 · patch 架构 = 子模块定义 + re-export + 成对 unpatch（CONFIRMED 强化）
- **发现**: 10 个 patch_* 定义于 instrument/_*.py 子模块、`__init__.py` 逐对 re-export（"from aegis.instrument._langchain import patch_langchain, unpatch_langchain"）；每框架成对 unpatch_*；`FrameworkPatch` 状态对象 + `InstrumentationState`（_state.py）
- **影响**: EK-G18"可逆补丁"强化——有 FrameworkPatch 状态对象与 InstrumentationState 支撑；README"12 frameworks"计数 = 10 框架 patch + 2 raw API（patch_anthropic/patch_openai），声明成立

### S-3 · autopolicy 三个生成器（CONFIRMED 强化）
- **发现**: autopolicy_llm.py 有 3 处 generate（OpenAI:181 / Anthropic:257 / 339 行第三生成器）
- **影响**: EK-G06 补"3 个 generator"（第三个类型未深读，标注）

## 3 个错误/风险点（Auditor 主动攻击）

### E-1 · "12 frameworks"初判异常 → 核实通过（无修正）
- **攻击**: `grep -c "def patch_" __init__.py` = 0——README 声称 12 frameworks 是否虚标？
- **核实**: patch 定义在子模块（10 个 _*.py）+ integrations/ 2 个 raw API patch——**12 声明成立**；EK-G02 补"patch 定义在子模块、__init__ re-export + 成对 unpatch"细节

### E-2 · EK-G16 空配置语义（PARTIALLY_CONFIRMED）
- **攻击**: `_setup_guardrails` cfg is None 时 return——"空配置跳过"；但 AEGIS_GUARDRAILS="default" 暗示存在**默认 guardrail 集**（未在 init 路径深读默认集的构成）
- **处理**: EK-G16 标注"默认 guardrail 集构成未深读"；行为（None→skip / 失败→warning）CONFIRMED

### E-3 · 测试徽章未独立复跑（MISSING 声明保留）
- **发现**: README 徽章（6400+ tests / 92% coverage）为 CI 自述；本 Audit 未复跑测试套件（时间预算）
- **处理**: 保留为 Coverage 缺口声明；测试文件 203 个 + conformance 断言已核实存在

## 遗漏（MISSING → 新增 EK-G19）
- **AEGIS_* 环境变量配置面**：AEGIS_INSTRUMENT/AEGIS_ON_BLOCK/AEGIS_AUDIT/AEGIS_GUARDRAILS 构成**第二配置入口**（env 族），与 YAML 策略配置并存——EK-G02 只提了接入梯度未系统覆盖 env 配置族 → 新增 EK-G19（env 配置面 + fail-closed 默认 on_block="raise"）

## Benchmark case（供 skill CI Gold Record）
- **BC-AEGIS-1**: fail-safe policy——destructive action 无规则匹配 → Approval.BLOCK（conformance test_agp_level1.py:99 断言可复跑）
- **BC-AEGIS-2**: 单调约束——delegation chain trust 非递增验证（action_claim.py:109-121 纯函数可复现）

## 结论
- 12 项抽查 11 CONFIRMED / 1 PARTIALLY；0 CONTRADICTED；主 Package 核心机制全部确认
- 修正项：MISSING×1（新增 EK-G19 env 配置面）+ 强化×3（S-1/S-2/S-3）→ Reconciliation 落地
