# 01 Project Layer — SkillFortify 项目地图

> 全部事实来自 dbb5942 快照实际内容（`repo/`）。标号 Fx 对应 06_validation 的证据登记。

## 1. 项目身份

| 项 | 值 | 证据 |
|----|-----|------|
| name | skillfortify | pyproject.toml |
| version | 0.6.0 | pyproject.toml / CHANGELOG.md `[0.6.0] - 2026-08-05` |
| description | "Supply chain security scanner for AI agent skills. Supports 22 frameworks." | pyproject.toml |
| license | Elastic-2.0（**非 MIT**，候选卡记 MIT 为 discrepancy） | pyproject.toml / LICENSE |
| python | >=3.11 | pyproject.toml |
| 作者 | Varun Pratap Bhardwaj | pyproject.toml |
| 论文 | https://arxiv.org/abs/2603.00195（31 页，五证明 + 基准） | pyproject.toml / wiki |
| CLI 入口 | `skillfortify = "skillfortify.cli.main:cli"` | pyproject.toml `[project.scripts]` |
| 依赖 | click / rich / cyclonedx-python-lib / pyyaml；可选 sat(python-sat)/registry(httpx) | pyproject.toml |
| 构建 | hatchling | pyproject.toml `[build-system]` |

## 2. 顶层结构

```
repo/
├── pyproject.toml / README.md / CHANGELOG.md / LICENSE / SECURITY.md / AGENTS.md / CONTRIBUTING.md / ATTRIBUTION.md
├── src/skillfortify/          # 主源码
├── tests/                     # 182 个 pytest 文件
├── benchmarks/                # SkillFortifyBench 生成器 + 实验 harness
├── wiki/                      # 13 个设计文档（Formal-Foundations 等）
├── docs/                      # 4 个使用文档（asbom/commands/getting-started/skill-lock-json）
├── .claude/                   # 项目内 Claude Code skills（含 gitnexus）
└── .github/                   # CI workflows
```

## 3. 源码模块地图（src/skillfortify/）

| 模块 | 职责 | 关键文件 |
|------|------|---------|
| `cli/` | Click CLI：scan/verify/lock/trust/sbom/frameworks/dashboard/registry-scan | main.py, scan.py, verify.py, lock.py, trust_cmd.py, sbom_cmd.py, registry_cmd.py, dashboard_cmd.py |
| `parsers/` | 22 框架 → 统一 ParsedSkill（probe→parse 两阶段） | base.py（ParsedSkill/SkillParser ABC）、20+ 格式 parser |
| `core/analyzer/` | **StaticAnalyzer 三阶段**：能力推断（保守过度近似）→ 危险模式检测 → 声明vs实际核对 | engine.py, patterns.py（威胁模式库）, models.py（Finding/AnalysisResult/Severity） |
| `core/capabilities/` | **能力格 lattice**（NONE<READ<WRITE<ADMIN，join/meet/bottom/top）+ CapabilitySet（per-resource 取 join，POLA permits/violations） | levels.py, models.py |
| `core/threat_model/` | **DY-Skill 威胁模型**：DYSkillAttacker（intercept/inject/synthesize/decompose/replay）+ SupplyChain（author→registry→dev→env）+ 攻击分类学（AttackClass/ThreatActor/AttackSurface/AttackType A1-A13） | dy_skill.py, taxonomy.py, messages.py |
| `core/trust/` | TrustEngine：compute_intrinsic（加权）/compute_score（依赖 min 传播）/apply_decay（时间衰减）/update_with_evidence（证据更新）/score_to_level | engine.py, propagation.py, models.py |
| `core/dependency/` | 依赖图 + 约束 + 解析器（SAT 可选） | graph.py, constraints.py, resolver.py |
| `core/lockfile/` | Lockfile：SHA-256 完整性（SRI 风格）+ SOURCE_DATE_EPOCH 可重现时间戳 | lockfile.py, operations.py, models.py |
| `core/sbom/` | ASBOMGenerator（Agent SBOM，cyclonedx） | generator.py, models.py |
| `discovery/` | SystemScanner：有界树遍历发现 Claude/OpenClaw/MCP/IDE 配置 | system_scanner.py, ide_registry.py |
| `dashboard/` | HTML 安全报告生成 | generator.py, template.py |

## 4. 核心数据结构

| 结构 | 定义位置 | 语义 |
|------|---------|------|
| `ParsedSkill` | parsers/base.py | 22 框架统一中间表示：name/version/source_path/format/description/instructions/declared_capabilities/dependencies/code_blocks/urls/env_vars_referenced/shell_commands/raw_content |
| `Capability(resource, access)` | capabilities/models.py | 不可变（frozen=True）"资源×访问级别"对，直接实现 Dennis & Van Horn (1966) capability 概念，注释明确"防止 TOCTOU" |
| `CapabilitySet` | capabilities/models.py | per-resource 最多 1 个能力、取最高访问级别（lattice join）；permits/is_subset_of/violations_against |
| `AccessLevel` | capabilities/levels.py | NONE=0/READ=1/WRITE=2/ADMIN=3 + join/meet/bottom/top 类方法（含交换/结合/幂等/单位元性质注释） |
| `Finding`/`AnalysisResult` | analyzer/models.py | Severity(IntEnum) + attack_class + capability_violation；AnalysisResult.is_safe/max_severity |
| `SkillMessage`/`SupplyChain` | threat_model/messages.py | DY-Skill 的消息与供应链拓扑（author→registry→developer→environment） |
| `AttackClass`/`ThreatActor`/`AttackType` | threat_model/taxonomy.py | 攻击类（data_exfil/priv_esc/prompt_injection/dep_confusion/typosquatting/namespace_squatting...）、威胁角色（malicious_author/compromised_registry/supply_chain_attacker/insider_threat）、攻击类型 A1-A13（测试断言"exact per paper 32"） |
| `TrustSignals`/`TrustScore`/`TrustWeights` | trust/models.py | 四信号（provenance/behavioral/community/historical）+ 信任分（intrinsic/effective/level） |
| `Lockfile`/`LockedSkill` | lockfile/lockfile.py | SHA-256 完整性哈希 + 解析依赖映射 |
| `SkillSpec`/`RenderedSkill`/`RunMetadata` | benchmarks/generator/core.py | 基准生成器的规格/渲染/运行元数据 |

## 5. 核心状态

- **能力状态**：CapabilitySet 内每个资源的 AccessLevel（lattice join 保持最高权限）
- **信任状态**：TrustScore（intrinsic / effective / level L0-L3），effective = intrinsic × min(deps)，随时间衰减
- **DY 攻击者知识集**：DYSkillAttacker.knowledge（K 集合，单调不减）
- **锁文件状态**：Lockfile 中 skill 名 → LockedSkill（含 SHA-256 哈希）
- **扫描结果状态**：AnalysisResult（is_safe / findings / inferred_capabilities）
- **基准生成状态**：BenchmarkGenerator._generation_order（确定性迭代，seed=42）

## 6. 主要测试体系

- **182 个 pytest 文件**，`[tool.pytest.ini_options] testpaths=["tests"]`
- 目录：cli/（命令层）、core/（analyzer/capabilities/dependency/lockfile/sbom/threat_model/trust/benchmark_generator）、core/test_analyzer.py、core/test_pola_and_trust_hardening.py
- **测试命名即行为文档**（关键揭示）：
  - analyzer：`test_inline_interpreter_on_ordinary_tooling_is_not_critical`（双用途分级）、`test_inline_interpreter_carrying_a_payload_stays_critical`、`test_zero_width_space_obfuscation_is_detected`、`test_dev_tcp_reverse_shell_is_detected`、`test_base64_decode_long_flag_is_matched`、`test_capitalised_prose_is_not_an_environment_variable`（防误报）
  - benchmark_generator：`test_no_structural_feature_predicts_the_label`、`test_both_classes_draw_names_from_one_vocabulary`、`test_specimens_share_one_installation_layout`（anti-leakage）
  - trust：`test_decay_after_69_days_halves_trust`、`test_decay_after_230_days_drops_to_ten_percent`、`test_monotonicity_property_all_signals`、`test_negative_evidence_raises`
  - threat_model：`test_mapping_exact_per_paper_32`、`test_typosquatting_is_registry_dependent`
- **实验 harness**：benchmarks/experiments（e3_coverage/e4_resolution/e5_trust/e6_lockfile/e7_endtoend，`python -m benchmarks.experiments` → benchmarks/results/experiments.json）

## 7. 主要配置

| 配置 | 位置 | 内容 |
|------|------|------|
| 打包/依赖 | pyproject.toml | 版本/依赖/scripts/build/ruff(100 行) |
| 运行时 | SOURCE_DATE_EPOCH | lockfile 时间戳可重现（CHANGELOG 0.6.0） |
| 基准 | benchmarks/generator/config.py | 攻击类型/良性类/seed=42 |
| CI | .github/workflows/ | 测试 workflow（README badge 引用 ci.yml） |

## 8. 权限 / Policy / Governance 机制

| 机制 | 证据 |
|------|------|
| 项目内 AGENTS.md（GitNexus 代码智能） | "MUST run impact analysis before editing any symbol"、"NEVER edit without gitnexus_impact"、9441 symbols / 17297 relationships / 225 execution flows —— 编辑即治理（影响分析门） |
| SECURITY.md 披露策略 | 支持 0.1.x；漏洞范围明确（错误安全判定/能力推断绕过/锁文件完整性/供应链/恶意当安全）；范围外（第三方依赖/超大输入 DoS known limitation/物理访问）；48h 确认承诺；coordinated disclosure |
| CONTRIBUTING.md | 贡献流程（未深读，存在） |
| 能力声明核对（POLA） | capabilities/models.py violations_against → analyzer 三阶段之三 |
| _assert_safe_output_root | benchmarks/generator/core.py —— 基准生成防写越界（生成器自身的沙箱边界） |

## 9. 主要外部依赖

- 运行时：click、rich、cyclonedx-python-lib、pyyaml
- 可选：python-sat（SAT 解析）、httpx（registry 扫描）
- 开发：pytest、pytest-cov、hypothesis、ruff
- 生态参照（文档中）：ClawHavoc / CVE-2026-25253 / MalTool 数据集 / OWASP ASI04 / Agentic Skills Top 10（AST01-10）
