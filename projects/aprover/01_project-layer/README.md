# 01 · Project Layer — AProver 项目地图

> 全部事实来自仓库实际内容（README.md / PIPELINE.md / pyproject.toml / 代码结构）；commit b9314c7。

## 1.1 项目定位
- 名称：AProver（Agentic Prover for AI-Generated Code），首个 agent：BMC-Agent
- 一句话：LLM 驱动的形式化验证 agent 套件；BMC-Agent = *agentic model checking* 原型
- 参考论文：*Agentic Model Checking*（Sun, Liu, Kroening, Xue；arXiv:2605.21434, 2026）
- 论文 2605.21434 即实现所依据的参考；README 要求引用（Citation 节）
- License：4-clause BSD（LICENSE）
- 版本：0.1.0（pyproject.toml）；Python ≥3.12

## 1.2 架构
三层分工（README "How it works" + PIPELINE.md）：
1. **Agentic 层**（[L]）：Spec Generator（Phase 1，top-down 每函数 pre/post）、CEx Classifier（Phase 3 S1）、Spec Refiner（Phase 3b）、Realism Checker（Phase 3 S4）、Domain Analyzer（Pass 1.5）、Spec Quality（Phase 4）
2. **Conventional 层**（[C]）：tree-sitter 解析（Pass 1）、BMC Engine（Phase 2：CBMC/Kani/JBMC harness 生成 + 求解）、CEx Dedup、Feasibility（S2 真实 callee 内联重验）、Dynamic Validation（S3：GCC 编译运行）、Caller propagation（Phase 3b/3c）、Soundness Guard（over-refinement BMC 检查）
3. **证据层**：五档 confidence tier + artifacts 落盘（spec.json / cbmc_result.json / classification.json / bug_report.json / harness / propagation_events.json）

## 1.3 核心模块（bmc_agent/ 71 模块 73,867 行）
| 模块 | 职责 | 状态 |
|---|---|---|
| pipeline.py（AMCPipeline） | 主编排 + 属性类免疫 + 反馈循环集成 | always on |
| config.py（1,039 行） | 全部配置（env 或 dataclass 字段） | always on |
| llm.py（1,902 行） | LLM 客户端抽象（anthropic/openai/claude-code/codex）+ per-role provider | always on |
| spec_generator.py / spec_generator_v2.py | Phase 1：SCC 分层拓扑生成顺序 + dual-spec | always on |
| dsl_to_cbmc.py / harness_generator.py | spec-DSL → CBMC 构造；harness 合成 | always on |
| bmc_engine.py + cbmc.py/kani.py/jbmc.py/backends/ | Phase 2 solver 后端（按扩展名分派） | always on |
| cex_validator.py（2,778 行） | Phase 3：三态判定 + 精化 + Soundness Guard | always on |
| dynamic_validator.py（1,297 行） | S3：GCC 复现 + signal 捕获 | opt（--enable-dynamic-validation） |
| realism_checker.py（2,777 行） | S4：LLM 真实性审计 + witness-pattern 短路 | opt（--enable-realism-check） |
| bug_reporter.py | 五档证据分级 + realism 执行 | always on |
| soundness_policy.py（96 行） | **健全性决策权威** | always on |
| spec_evidence.py | v2 spec 生成调用者证据收集 | v2 路径 |
| feedback_loop.py / flag_selector.py / spec_quality.py | 自改进/旗标选择/规格质量 | opt |
| frama_c.py / acsl.py / jml_specs.py / loop_invariants.py | 反向规格合成（--oracle frama-c / specs-bench） | opt |

## 1.4 生命周期 / 控制流主链
```
源码 → Pass1 解析+全局调用图[C] → Pass1.5 Domain Analyzer[L] → Phase1 Spec[L]（SCC 分层+dual-spec）
→ Phase1.5 Flag Selector[L](opt) → Phase2 BMC[C]（k=4 unwind, 120s, threat-model 基线）
→ CEx Dedup[C] → Phase3 验证（S1 可达性[L+C] → S2 可行性[C] → S3 动态复现[C](opt) → S4 真实性[L](opt)）
→ REAL_BUG→报告 / SPURIOUS→Refiner[L]+Soundness Guard[C]→3c 自验+3b 调用者传播[C] / UNRESOLVED→跟踪跳过
→ Phase4 Spec Quality[L+C](opt) → BugReport（五档）
```

## 1.5 核心数据结构/状态
- Spec（pre/post/类别/JSON reasoning；DSL 谓词：valid/valid_string/valid_range/in_bounds/null/owns/locked）
- Counterexample（failing_property/trace）；CExOutcome 三态
- ValidationResult（outcome + reachability/feasibility error 标志 + is_latent_bug）
- RealismVerdict / RealismCheckResult；BugReport（confidence 五档）
- Justification/Action（soundness_policy 枚举）
- 状态机：函数级 pending→spec'd→bmc'd→classified→refined→(clean|bug|unresolved)

## 1.6 测试体系
- 121 测试文件 / 2,410 test 函数（pytest，testpaths=["tests"]）
- 本机抽样（soundness/realism/tier/evidence）：213 passed / 1 failed / 2 skipped / 2,220 deselected
- 失败：test_agentic_components.py::test_agentic_keeps_classifier_on_realism_tools_on_triage_off_dynval_on（期望 --agentic 下 realism=True，实现 False）→ 测试-实现-文档漂移

## 1.7 配置 / 权限与治理
- 配置 = 环境变量 ↔ Config dataclass 字段（unwind=4 / timeout=120 / max_refinement=5 / dual_spec=true / per-role provider 等；见 README Configuration 全表）
- 治理：soundness_policy.py = 删除/降级判定的单一权威源；PIPELINE.md = 实现的 ASCII 自述；PLAN_autonomous_mode.md = 自主模式计划
- 外部依赖：CBMC（必需）/ Kani（Rust）/ JBMC+JDK（Java）/ Frama-C+Alt-Ergo（oracle）/ ANTHROPIC_API_KEY 或本地 claude/codex CLI

## 1.8 重要设计 / ADR 证据
- ADR 类证据 = 仓库内文档 + 代码注释中的决策痕迹：
  - realism_checker 注释记录各短路模式的来源事件（jq jv stub-disconnect 2026-05-13、ch341 NULL-guard 2026-05-18、pl2303 USB-serial 2026-05-18、vibeos vacuity 2026-06-12）——模式库随实测失败演进
  - soundness_policy 模块 docstring 即 ADR（删除不对称性论证）
  - PIPELINE.md 明确 UNRESOLVED"never silently dropped"（cex_validator docstring）
  - findings/ SESSION_SUMMARY 系列 = 研究日志（2026-05-21 起持续）

## 1.9 实测结果（findings/）
- VibeOS：37 内核模块 / 675 函数 → 13 realistic bugs（realism 滤掉 48 个不真实反例）；动态复现 + confirmed_system_entry 缺陷 + CWE-190 calloc 溢出（交叉验证 VibeOS issue #26）
- llm.c：22/30 函数 clean（M1-M2 里程碑，up from 4/30）；首次 BMC 应用于真实 ML 训练程序
- linux_drivers：8 个 mainline Linux driver findings（HEAD commit）
- libarchive / nghttp2 / oss-curl / llama_cpp_ggml / aws_neuron_driver：sweep 记录
