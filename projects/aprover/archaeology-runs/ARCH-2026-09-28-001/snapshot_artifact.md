# Repository Snapshot — AProver（ARCH-2026-09-28-001）

## 快照锚点

| 项 | 值 |
|---|---|
| 仓库 | https://github.com/agentic-prover/aprover.git |
| commit SHA | **b9314c704c1142d195ae43abde4ad9e03f84aaa8** |
| branch | main（HEAD = origin/main） |
| 版本 | aprover 0.1.0（pyproject.toml `version = "0.1.0"`） |
| commit 时间 | 2026-09-15 16:01:17 +0400（"docs: add eight mainline Linux driver findings from BMC-Agent"） |
| 克隆时间 | 2026-09-28（浅克隆 depth=1） |
| 规模 | 268 Python 文件 / 767 总文件 / bmc_agent/ 73,867 行 |
| License | 4-clause BSD（LICENSE） |

## 项目定位

**AProver — Agentic Prover for AI-Generated Code**：一套 LLM 驱动的形式化验证 agent 套件。首个 agent **BMC-Agent** 是 *agentic model checking* 原型——LLM agent（spec 生成/反例分类/spec 精化）+ 健全有界模型检查后端（CBMC/Kani/JBMC）配对；agent 负责语义推理，solver 在展开界内提供形式保证。参考论文：*Agentic Model Checking*（Youcheng Sun, Jiawen Liu, Daniel Kroening, Jason Xue, arXiv:2605.21434, 2026）。

**设计原则**："agents propose, conventional tools dispose"——LLM 产生的每个健全性相关决策必须先通过传统检查（CBMC 查询/SMT 健全性守卫/运行时确认）才影响验证结论。

## 项目基础地图

### 语言与入口
- Python ≥3.12；入口：`aprover.cli:main`（薄壳 25 行）与 `bmc_agent.cli:main`（核心 CLI）；`pyproject.toml` 注册两个 script 入口。
- 依赖：anthropic / tree-sitter(+c/rust) / rich / pydantic；optional web extra：fastapi/uvicorn/httpx/tiktoken。

### 顶层结构
```
aprover/          薄壳 CLI（25 行，转发到 bmc_agent）
bmc_agent/        核心（73,867 行，71 模块）——管线/验证/LLM/解析/后端/评估
actions/          辅助动作脚本
examples/         合成 + 真实目标（simple_driver/sensor_hub/block_device/cross_file_demo/vibeos 内核）
tests/            121 测试文件 / 2410 test 函数
findings/         17 组实测发现（vibeos/llm_c/linux_drivers/libarchive/nghttp2/oss-curl 等）
integrations/     Claude Code / Codex 等 CLI 集成
web/              FastAPI 聊天前端（aprover.ai）
nix/              开发环境（tools.nix：CBMC/Kani/JBMC 源码构建）
docs/ PIPELINE.md PLAN_autonomous_mode.md README.md
```

### 核心模块（bmc_agent/）
| 模块 | 职责 |
|---|---|
| `pipeline.py`（AMCPipeline） | 主编排：parse→spec→BMC→CEx 验证→精化循环（CEGAR）；跨文件调用图两遍构建；属性类免疫 |
| `spec.py` / `spec_generator.py` / `spec_generator_v2.py` / `spec_refiner.py` | Spec 模型 + Phase 1 生成（top-down、dual-spec 双重生成+分歧标记）+ 精化 |
| `dsl_to_cbmc.py` / `harness_generator.py` / `agentic_harness_gen.py` | spec-DSL（valid/valid_string/valid_range/in_bounds/…）→ CBMC 构造；harness 合成 |
| `bmc_engine.py` / `cbmc.py` / `kani.py` / `jbmc.py` / `frama_c.py` / `backends/` | Phase 2 solver 后端（按扩展名自动分派：C→CBMC、Rust→Kani、Java→JBMC；`--oracle frama-c` 反向合成） |
| `cex_validator.py` | Phase 3：反例验证（concretize→REAL_BUG/SPURIOUS/UNRESOLVED）；S1 reachability（BMC 子查询 + LLM fallback）、S2 callee feasibility、精化 + Soundness Guard、向上传播 |
| `dynamic_validator.py` | Phase 3 S3：GCC 编译运行复现 harness，signal handler 捕获 SIGSEGV/SIGABRT/SIGFPE/SIGILL |
| `realism_checker.py` | Phase 3 S4：LLM 审计每个 REAL_BUG 的可利用真实性；witness-pattern 确定性短路（uninitialized lib / jv stub-disconnect / NULL-guard violation 等） |
| `bug_reporter.py` | 五层证据分级（confirmed_dynamic / confirmed_system_entry / confirmed_bmc / likely / unlikely） |
| `soundness_policy.py` | **健全性策略单一事实源**：deterministic_verifier/self_verifying_witness 可 DELETE，agentic_judgment 只能 RETIER |
| `llm.py`（1,902 行） / `llm_judge.py` / `llm_tool_loop.py` | LLM 客户端抽象 + 判定 + 工具循环 |
| `spec_evidence.py` | v2 spec 生成的调用者证据收集（harvest_callers：调用点/地址取用点/测试路径过滤） |
| `spec_quality.py` | Phase 4：变异/覆盖/一致性检查（可选） |
| `feedback_loop.py` / `flag_selector.py` / `domain_analyzer.py` / `preprocessor.py` | 可选阶段：自改进循环 / 每函数 CBMC 旗标选择 / 领域分析 / 预处理 |
| `acsl.py` / `jml_specs.py` / `loop_invariants.py` / `global_invariants.py` / `universal_contracts.py` | ACSL/JML 输出、循环不变量、全局不变量、通用契约 |
| `evaluation/` | 基线与评估 |

### 核心数据结构/状态
- `Spec`（dataclass：pre/post/类别/JSON reasoning；DSL 谓词集合）；`ParsedSpec`（双解析比对）
- `Counterexample`（failing_property/trace/…）；`CExOutcome`（REAL_BUG/SPURIOUS/UNRESOLVED）
- `ValidationResult`（outcome + reachability/feasibility error 标志 + is_latent_bug）
- `RealismVerdict`（REALISTIC/UNREALISTIC/UNCERTAIN）；`RealismCheckResult`
- `BugReport`（confidence 五档 + reasoning trail）
- `Justification`/`Action`（soundness_policy 枚举）
- 管线状态：函数级验证隔离（callee 用 LLM postcondition 的 `__CPROVER_assume` stub）→ 精化后 3c 自验 + 3b 调用者传播（compositional）

### 测试体系
- pytest：121 文件 / 2,410 test 函数；实测抽样（soundness/realism/tier/evidence 相关 213 passed / 1 failed / 2 skipped）
- **实测失败**：`test_agentic_components.py::test_agentic_keeps_classifier_on_realism_tools_on_triage_off_dynval_on` —— 期望 `--agentic` 下 `enable_realism_check=True`，实现返回 False（README §Agentic mode 明确 realism/triage 默认 OFF 且独立 opt-in）→ **测试-实现-文档三方漂移**（测试未随 README/实现更新），实现符合文档

### 配置/权限与治理机制
- 全部设置 = 环境变量 或 `Config` dataclass 字段（`config.py` 1,039 行；如 `BMC_AGENT_CBMC_UNWIND=4`、`BMC_AGENT_MAX_REFINEMENT_ITERS=5`、`BMC_AGENT_ENABLE_DUAL_SPEC=true`、`BMC_AGENT_CBMC_TIMEOUT=120`、provider 每角色覆盖 `BMC_AGENT_LLM_<ROLE>_PROVIDER`）
- 治理：`soundness_policy.py` 是删除/降级判定的**单一权威源**；`PIPELINE.md` 是实现的 ASCII 自述；`PLAN_autonomous_mode.md` 描述自主模式
- 外部依赖：CBMC（PATH）、Kani（Rust 可选）、JBMC+JDK（Java 可选）、Frama-C+Alt-Ergo（oracle 可选）、ANTHROPIC_API_KEY（或本地 claude/codex CLI）

### 实测发现（findings/）
- VibeOS（37 内核模块 / 675 函数）：确认 **13 个真实 bug**（realism 过滤掉 48 个不真实反例）；动态复现崩溃（net_get_mac 空解引用 / stbtt__h_prefilter 栈越界写 / stbtt_PackEnd double-free）+ confirmed_system_entry 缺陷（vfs_lookup 深路径栈溢出 / vfs_open_handle strcpy 溢出 / vfs_close_handle UAF）；calloc 整数溢出（CWE-190）交叉验证 VibeOS issue #26
- llm.c（train_gpt2.c ~1100 行）：M1-M2 里程碑（--infer-field-validity / --infer-array-param-bounds / --scale-down / --safety-only）下 **22/30 函数 clean**（up from 4/30）；首次将 BMC 应用于真实 ML 训练程序
- linux_drivers：8 个 mainline Linux driver findings（HEAD commit 主题）
- libarchive / nghttp2 / oss-curl / llama_cpp_ggml / aws_neuron_driver 等 sweep 记录

### 运行观察（本机实测）
- soundness_policy 行为验证：DETERMINISTIC_VERIFIER→DELETE、SELF_VERIFYING_WITNESS→DELETE、AGENTIC_JUDGMENT→RETIER；refiner（CBMC 排除 ∧ 调用点确定性检查）→DELETE 否则 RETIER；witness 精确复现→DELETE 否则 RETIER —— 与文档一致
- pytest 抽样：213 passed / 1 failed（见上）/ 2 skipped

## 事实可追溯性
以上全部条目可直接定位到仓库文件：README.md（定位/管线/配置/实测）、PIPELINE.md（管线 ASCII）、soundness_policy.py（健全性规则）、realism_checker.py（短路模式）、cex_validator.py（三态判定）、dynamic_validator.py（信号捕获）、bug_reporter.py（五档分级）、pyproject.toml（版本/入口/依赖）、findings/（实测结果）、tests/（测试量 + 实测漂移）。
