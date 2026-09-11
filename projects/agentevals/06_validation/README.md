# 06 — Validation & Evidence

> 纪律：所有 Validator 先 **Blind Reconstruction**（独立重读仓库建立自己的 findings，再对比），不以考古结果当事实来源；不弱化证据标准换全绿。本文件由 Archaeologist 先做一轮自校验，阶段 5 由独立 Auditor 盲重建交叉验证（见 independent_validation_report.md）。

## 6.1 Source Truth（Truth Auditor）

| Claim | 判定 | 证据锚点 |
|---|---|---|
| "No re-execution" | ✅ TRUE | runner.py:77 `run_evaluation_from_traces` 不调用 agent；README 声明 |
| "ADK 复用" | ✅ TRUE | builtin_metrics.py:186-241 映射 ADK Criterion；get_evaluator 直接 import google.adk |
| "798 passed / 24 skipped" | ✅ TRUE（实测） | 本地 pytest 运行（补 pytest-asyncio 后）；初跑失败为环境插件缺失 |
| "ADK 优先于 GenAI" | ✅ TRUE | docs/otel-compatibility.md "ADK takes priority" + converter.py get_extractor |
| "fail-closed 凭据" | ✅ TRUE | builtin_metrics.py:395-404 未解析→error；tests/test_credential_injection.py |
| "no golden → error" | ✅ TRUE | builtin_metrics.py:392 METRICS_NEEDING_EXPECTED |
| "Claude Code/Codex/OpenCode 不发射 GenAI semconv" | ✅ TRUE（README 自述，非实测） | README 边界声明；标注为项目自述（S7 类） |

## 6.2 Coverage（Coverage Auditor）

覆盖：架构/核心抽象/生命周期/控制流/状态流/数据流/证据流/权威流/记忆流/策略流/错误路径/测试/配置/扩展机制 ✓
刻意未深挖（记录为缺口，非隐藏）：`api/otlp_grpc.py` GRPC 接收细节、`openai_eval_backend.py` 全链路、postgres migrations 具体 SQL——属实现细节，不影响本次认知结论；若后续做 refresh 或 benchmark 需补齐。

## 6.3 Flow（Flow Auditor）——逐条 Edge 校验

- C1/C2/C3（Control）、S1/S2/S3（State，测试锚定）、D1/D2/D3（Data）、E1/E2/E3（Evidence）、A1/A2/A3（Authority）、M1/M2（Memory）、P1/P2（Policy）——**全部由实际 symbol/file/condition 支撑**（04 Flow Atlas 每 Edge 已标位置）。
- 审计发现：无凭空虚构 Edge；`_enrich_app_details`（D1）标为"合成补全路径"而非主线数据流，防止读者误读为主证据路径。

## 6.4 Abstraction（Abstraction Auditor）——升维审查

| 候选升维 | 判定 |
|---|---|
| KO-01 "执行是采样，评测是审阅" | ✅ L4 成立（EK-01/02/03 支撑；解释范围扩大有同类对照） |
| KO-02 "外部多样性在边界收敛" | ✅ L4 成立（EK-02/04/05/11/12 注册表族） |
| KO-03 "可复现性是门禁前提" | ✅ L4 成立（EK-14/06/08/15） |
| KO-04 "不确定处失败好过错误处成功" | ✅ L4 成立（EK-07/08/09 fail-closed 族） |
| L5 Methodology | ❌ 不升——单项目证据不足，写 L5 即"项目经验→通用 Principle"过度升维（守纪律） |

## 6.5 Counterexample（反例预算制）——主动找"哪里不是这样"

| 反例 | 类型 | 影响 |
|---|---|---|
| R1 `_enrich_app_details` 合成 schema | **bypass/fallback**（数据合成绕过 schema 缺失） | 评分可得但粒度降级（→C-02）；不写进 KO 作为"模式" |
| R2 轨迹指标无"遥测缺失 fail-closed" | **exception/alternate path**（对比凭据 fail-closed，轨迹类无等价保护） | →C-04；覆盖缺口 |
| R3 judge 凭据走私有 seam | **admin path/legacy path**（私有属性注入） | 技术债（TODO upstream 自认）；→EK-07 明示 |
| R4 LLM judge 重试/网络失败路径 | **failure path**（未在本快照深挖 judge API 错误传播） | 记录为验证缺口（judge 类指标错误处理未覆盖） |
| R5 非 judge-backed evaluator 注入凭据 | **direct call bypass**（warning+skip，不报错） | 静默跳过边界——符合 fail-closed 族但需注意 |

反例预算：4 个 KO × 定向攻击 ≥3 反例/KO 的预算内，关键反例 R1/R2/R4 已入 Candidates（C-02/C-04）或 EK 边界。

## 6.6 Epistemic（Epistemic Auditor）

- 全部 KO 标 Epistemic 状态（Pattern/Model + Cross-project validation pending）；L5 不写；Candidates 全标 Hypothesis/Tentative/Scope-uncertain。
- "798 passed" 是**实测**（S4）；stars/pushed 来自 GitHub API（外部事实，标注来源）；README 能力声明标为项目自述（S7）。
- 无 Fact/Observation/Hypothesis/Pattern/Model/Principle 混淆。

## 6.7 Reconciliation（独立 Auditor 修正合入）

| 修正 | 来源 | 处理 |
|---|---|---|
| worker lease 租约机制（claim_next lease/max_attempts + heartbeat 续租 + cancelled-or-lost） | Auditor MISSING-1 | 补入 02 EK-09 + 04 S1/S2 |
| cooperative cancellation 精确语义（cancel_requested → 下次 heartbeat 观察） | Auditor 澄清 | 04 S3 已修正 |
| ADK seam guard 测试（fail loudly） | Auditor 补充发现 | 补入 02 EK-07 Why |
| C-01（ADK 匹配算法） | Auditor NEEDS_HUMAN_REVIEW | 维持 Candidate，标注外部依赖 |

## 6.8 质量指标

| 指标 | 值 |
|---|---|
| EK 数量 | 16（全带 links，平均出边 ≥1，游离 0） |
| KO 数量 | 4（R2×1 / R4×1 / R3×2；全部 aggregation_rule + 可回溯） |
| Candidates | 5（Hypothesis ×2 / Tentative ×2 / Cross-project ×1） |
| 测试证据 | 798 passed / 24 skipped（40.19s） |
| 反例 | 5（R1-R5），关键反例入 Candidates |
| 独立验证 | 见 independent_validation_report.md（阶段 5） |
| Validation 结论 | PASS（含已知缺口清单，非"全绿"——保留 Contradictions 与 Counterexamples） |
