# Repository Snapshot — ARCH-2026-10-22-001（agent-aegis / Acacian/aegis）

## 快照事实（可回溯浅克隆实盘）
- **repository**: https://github.com/Acacian/aegis.git
- **HEAD**: `134580e37190ed9297a245eb3c902972ca9bb334`（main，"fix: unpin drift broke typecheck; pin the linters and narrow a YAML load"，2026-08-29 23:36:41 +0900）
- **version**: 1.0.0（pyproject.toml `version = "1.0.0"`；CHANGELOG.md:58 `[1.0.0] — 2026-08-09`；tags 至 v1.0.0，浅克隆未含完整 tag 历史）
- **活跃度**: GitHub API pushed_at 2026-10-16（考古前 6 天，tag/branch push）；created 2026-03-20；★18；open_issues 6
- **规模**: 667 文件 / 449 py / src 71,670 行 / 测试文件 203 个（tests/ + packages/*/tests）；README 徽章：6400+ tests、coverage 92%、OpenSSF Best Practices 参与

## 基础地图
| 维度 | 事实 |
|---|---|
| 语言/运行时 | Python 3.11+（pyproject target-version py311）；包名 `agent-aegis`；核心依赖仅 `pyyaml>=6.0`，其余全部 optional（playwright/langchain/crewai/google-adk/pydantic-ai/openai-agents/anthropic/httpx/mcp/starlette+uvicorn/redis/asyncpg） |
| 定位 | "The governance layer for AI agents. One API, 12 frameworks, every governance primitive"——统一 prompt-injection 阻断/PII 掩码/策略执行/信任委托/防篡改审计；`pip install agent-aegis` → `aegis.auto_instrument()` |
| 入口 | `src/aegis/__init__.py`：惰性加载设计（import ~30ms；仅 Aegis/init/shutdown/get/auto_instrument 急切加载）；`src/aegis/init.py`（Aegis.init：config 发现→激活→audit/guardrails/cost/integrations）；`src/aegis/instrument/__init__.py`（10 框架 patch_* + 2 raw API；`AEGIS_INSTRUMENT=1` 环境变量零代码接入） |
| 核心模块 | adapters/（anthropic/crewai/langchain/mcp/openai_agents/playwright/httpx + base）；core/ 100+ 模块（policy 引擎 / action_claim 审批门 / merkle_audit / autopolicy / rbac / sandbox_policy / circuit_breaker / cascade_guard / cross_tool_privacy / data_isolation / taint / hazard_classifier / mcp_* 全家桶：stdio_guard/shadow/rate_limiter/response_scanner/threat_intel/vuln_db…）；cli/（check/scan/score/plan/diff/monitor/compliance/test_cmd）；contrib/pydantic_ai；独立包 packages/{crewai,langchain,openai-agents}-aegis |
| 核心数据结构 | `Policy/PolicyRule/PolicyDecision`（core/policy.py：evaluate + 256 缓存 + from_yaml/from_yaml_files/merge）；`Action/ActionClaim/ClaimVerdict/ImpactVector`（action/action_claim.py：declared vs assessed fields + 单调约束 validate_monotone_constraint + delegation chain）；`MerkleAuditTree/MerkleLeaf/MerkleProof`（merkle_audit.py：SHA256 规范 JSON 叶哈希 + 批证明）；`AegisConfig`（config.py）；Aegis singleton（init/shutdown 状态机） |
| 测试体系 | tests/（cli/ + conformance/：`test_agef_schema.py`、`test_agp_level1.py`）+ packages/*/tests；`core/policy_test_suite.py` + `policy_test_runner.py`（策略即测试）；benchmarks/ |
| 配置 | `policy.example.yaml`（version "1"：defaults risk_level/approval + rules：match type→risk_level/approval + conditions 如 bulk>100 升级）；`policy.schema.json`；`aegis-server.example.yaml`；server.json/smithery.yaml（MCP server 部署）；Dockerfile；action.yml（GitHub Action 接入） |
| 权限与治理机制 | Policy evaluate→PolicyDecision（approval: auto/approve）；ActionClaim 审批链（ImpactVector 单调约束防降级授权）；RBAC；sandbox_policy；circuit_breaker/cascade_guard；cross_tool_privacy/data_isolation/taint（数据面）；cost_policy/budget/rate_limiter（成本面）；merkle_audit（防篡改审计）；otel_export（可观测） |
| 开放标准（自述） | specs/agef/v1（AGEF schema.json，Draft 2026-03-24："AI governance 的 SARIF/CEF"）；specs/agp/v1/protocol.md（AGP v1，Draft："MCP 标准化通信侧，AGP 标准化治理侧"）；conformance 测试钉住 schema |
| 治理元机制 | GOVERNANCE.md（ADR 以 `adr` label issue 承载；breaking 需 Discussion + CHANGELOG）；SECURITY.md；CODE_OF_CONDUCT.md；docs/security/ |
| CVE 语境 | 本周 Harness 事件集群（Archon CVE-2026-111322 控制面配置缺失）对照组：aegis 属"运行时中间件防御"侧（进程内插桩+策略），非控制面配置类 |

## 考古焦点（锁定 KnowlegeMap 卡三主张）
1. **Pillar 1** runtime guardrails（PII/injection）：init.py `_PIIGuardrailAdapter`/`_InjectionGuardrailAdapter` + core/guardrails 实现
2. **Pillar 2** YAML 声明式策略引擎 + approval gates：core/policy.py + core/action_claim.py + policy.example.yaml
3. **Pillar 3** AGEF/AGP 开放标准：specs/ 自述（Draft）——**"厂商自述无第三方采纳证据"为 Epistemic 审计靶点**
