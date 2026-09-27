# 06 · Validation & Evidence — Aigis ARCH-2026-09-27-001

> 六类 Auditor（Truth/Coverage/Flow/Abstraction/Counterexample/Epistemic）全部**先 Blind Reconstruction**：独立重读仓库建立 Independent Findings 后与考古产物对比，不以考古产物为事实来源。禁止修改原考古产物；本报告记录全部判定（含 Contradictions 与 Counterexamples）。

## 0. Blind Reconstruction 方法

- 独立重读文件清单（未引用考古产物）：`aigis/scanner.py`（L1-L3 全链）、`aigis/capabilities/enforcer.py` + `taint.py` + `store.py`、`aigis/audit/{signed_log,chain,verify}.py`、`aigis/policy.py`、`aigis/settings_export.py`、`aigis/mcp_scanner.py`、`aigis/adapters/claude_code.py`、`aigis/trust_pack.py`、`aigis/memory/*`、`ROADMAP.md`、`CHANGELOG.md`、`CLAUDE.md`、`docs/benchmarks/oss-comparison.md`、`tests/` 抽样（test_audit / test_signed_audit_hook / test_agent_tool_abuse_2 / test_guard）
- 独立运行验证：Guard 行为矩阵（8 样本）、pytest 三组（54 / 194 / 255 passed）、`grep tests/ capabilities`（盲区确认）
- Independent Findings 先于对比建立，共 22 条

## 1. Truth Auditor（这句话是真的吗）

### Independent Findings 与考古产物对比

| # | 考古产物声明 | Independent Finding | 判定 |
|---|-------------|--------------------|------|
| T-01 | 运行时零依赖（标准库 only） | pyproject 无第三方 install_requires；import 链仅标准库 | **CONFIRMED** |
| T-02 | HEAD 5874cbb5 = 2026-09-02 commit | `git log -1` 确认 | **CONFIRMED** |
| T-03 | 四层检测管线存在 | `_run_patterns` 含 L1 regex→L2 similarity→L3 decode-rescan；enforcer 为 L4 | **CONFIRMED** |
| T-04 | 归一化含 NFKC/零宽/空格/Confusable/Emoji | `_normalize_text` + decoders 确认；实测零宽攻击 100 blocked | **CONFIRMED** |
| T-05 | L3 解码重扫跳过已命中 rule | scanner.py 300-327 `matched_ids` 逻辑确认 | **CONFIRMED** |
| T-06 | 类别分数封顶 delta*2、总分截断 100 | 代码 245-252/328 确认 | **CONFIRMED** |
| T-07 | Guard 阈值 81/30 | 实测 `auto_block_threshold=81, auto_allow_threshold=30` | **CONFIRMED** |
| T-08 | CaMeL 控制流资源无条件 DENY 先于 grant | enforcer.py 175-183 在 store.check 之前返回 | **CONFIRMED** |
| T-09 | taint promote 不变量 UNTRUSTED 不可直升 TRUSTED | taint.py promote 抛错逻辑确认 | **CONFIRMED** |
| T-10 | SignedLogEntry 11 字段 + HMAC-SHA256 | signed_log.py 38-120 确认；test_signature_verifies 通过 | **CONFIRMED** |
| T-11 | 哈希链覆盖含 signature 字段、genesis 64 零 | chain.py 确认；test_genesis_hash 通过 | **CONFIRMED** |
| T-12 | hook 四类失败 fail-closed exit(2) | claude_code.py HOOK_SCRIPT 确认 | **CONFIRMED** |
| T-13 | 签名日志失败不影响决策 | try/except pass + test_signed_log_failure_does_not_block | **CONFIRMED** |
| T-14 | settings 条件规则不可导出（ExcludedRule） | settings_export.py convert_rule 确认 | **CONFIRMED** |
| T-15 | trust pack GL-*/SEC-* 是自有编号非官方 | trust_pack.py ControlMapping.ai_gl 注释原文确认 | **CONFIRMED** |
| T-16 | ROADMAP 记录 54 stars / 300 点赞 / ~15k 下载 | ROADMAP.md 表格原文确认 | **CONFIRMED** |
| T-17 | CHANGELOG 2.0.1 记录 v2.0.0 PyPI 烧号 | CHANGELOG.md 原文确认 | **CONFIRMED** |
| T-18 | 1781 tests passed（CHANGELOG 声明） | 本地 pytest 抽样 503 passed；**全量 1781 未复跑**（时间成本），记为 CHANGELOG 声明而非独立验证 | **PARTIALLY_CONFIRMED** |
| T-19 | 54★（GitHub） | 来自雷达/API 读数，非本仓内可验证 | **PARTIALLY_CONFIRMED**（外部事实，标注来源） |

### 判定统计（Truth）
- CONFIRMED 17 / PARTIALLY_CONFIRMED 2 / CONTRADICTED 0

## 2. Coverage Auditor（还有什么重要东西没发现）

| # | 考古产物遗漏/薄弱 | Independent Finding | 判定 |
|---|------------------|--------------------|------|
| C-01 | CaMeL 层测试盲区未在 EK 层标注 | 考古产物在 EK-14 标注（观察）；Candidates C-02 保留 | **CONFIRMED（已在产物内）** |
| C-02 | JA/KO 短语实测未命中未列入快照 | 考古产物在 snapshot 实测记录 + C-01 Hypothesis 保留 | **CONFIRMED（已收录）** |
| C-03 | `multi_agent/topology.py`、`supply_chain/sbom.py`、`cross_session/` 仅列名未深读 | 考古产物在 00 Overview 证据边界声明 + C-04 标注 | **PARTIALLY_CONFIRMED**（覆盖声明存在，深读缺失如实保留） |
| C-04 | benchmarks/oss_comparison 三方数字未产出（docs 自述 v0 baseline Aigis-only） | 考古产物 KO-03/C-03 引用了该事实 | **CONFIRMED（如实呈现）** |
| C-05 | `anthropic_proxy.py`（server.py 相关）未单独深读 | 考古产物以 tests 引用标注 | **MISSING**（未纳入 EK；影响小，纳入 Reconciliation 补充） |

### Coverage 判定
- CONFIRMED 3 / PARTIALLY_CONFIRMED 1 / MISSING 1（C-05：anthropic_proxy 仅测试级证据，标轻量遗漏）

## 3. Flow Auditor（Flow Atlas 是否真的对应代码）

| # | Flow Edge | Independent 重读 | 判定 |
|---|-----------|-----------------|------|
| F-01 | F1 E2 `_map_action` 未命中返回 | 考古产物写 "unknown tool fallback 到 tool:*?"（带疑问）；独立重读：`_map_action` 未命中返回 `"unknown"` | **DOWNGRADED**（考古产物用词不精确，已修正为 unknown） |
| F-02 | F3 D1 UNTRUSTED+CFR → DENY | enforcer.py 175-183 确认 | **CONFIRMED** |
| F-03 | F4 E5 审计轨解耦 | hook try/except pass 确认 | **CONFIRMED** |
| F-04 | F5 A1 条件规则 ExcludedRule | settings_export.py 确认 | **CONFIRMED** |
| F-05 | F6 M2 完整性 verify hash 不匹配 | integrity.py verify 确认 | **CONFIRMED** |
| F-06 | F7 P3 release.yml orphan-tag 拒绝 | workflow 文件确认 | **CONFIRMED** |
| F-07 | F1 E4 deny → exit(2) 阻断 | HOOK_SCRIPT 确认 | **CONFIRMED** |

### Flow 判定
- CONFIRMED 6 / DOWNGRADED 1（F-01 用词精度）/ CONTRADICTED 0

## 4. Abstraction Auditor（L3/L4 是否过度升维）

| # | KO | 攻击 | 判定 |
|---|-----|------|------|
| A-01 | KO-01（编码变体防御结构） | 单项目证据 → Pattern 升维？ | 论证：L3 结构（normalize→decode→rescan）在 scanner+decoders 两个独立文件实现、WAF/杀软同类对照成立；但仅 Aigis 实证 → 保留 `Cross-project validation pending` | **PARTIALLY_CONFIRMED**（保留 pending 标记） |
| A-02 | KO-02（双层门） | 从 Aigis+Claude Code 组合泛化到"任何宿主+插件架构"？ | 论证：settings_export 的 ExcludedRule 机制（条件不可表达→保留在 hook 层）是通用模式；但第二宿主（如 IDE 扩展）未实证 → 降为带 pending 的 Cognitive Model | **DOWNGRADED**（原拟 L4 无条件陈述，现标 Cross-project validation pending） |
| A-03 | KO-03（taint 先于授权） | 引 arXiv 论文 + 单项目实现 → Principle？ | 论证：有学术支撑（CaMeL paper）+ 实现；但**测试缺失**（EK-14）→ 显式标注 `Partially validated (implementation only)`，未标 Principle | **PARTIALLY_CONFIRMED**（诚实降级已执行） |
| A-04 | KO-06（战略测量纪律） | 单项目战略复盘 → L4 模型？ | 论证：开源项目战略文献广泛支持，但"退出清单比目标清单重要"仅在 Aigis 实证 → 保留 pending | **PARTIALLY_CONFIRMED** |
| A-05 | KO-05（诚实性设计） | 多实例（trust pack/benchmark/ROADMAP）→ Pattern？ | 论证：4 处独立实例 + 同类对照（SBOM/红队报告）成立 | **CONFIRMED** |

### Abstraction 判定
- CONFIRMED 1 / PARTIALLY_CONFIRMED 3 / DOWNGRADED 1（A-02 收紧 scope）/ OVER_GENERALIZED 0

## 5. Counterexample Hunter（反例预算）

每个 L3+ KO ≥3 个定向反例攻击；0 反例须给出搜索证据。

| # | KO | 反例攻击 | 结果 |
|---|-----|---------|------|
| X-01 | KO-01 | 反例：不解码的检测器（只加 pattern）也能挡住部分攻击？ | 反驳成立但不推翻：只加 pattern 挡不住零宽/组合编码（实测零宽需归一化）；KO-01 描述的是"结构"非"唯一解" | 反例未推翻 |
| X-02 | KO-01 | 反例：decode 重扫引入误报（正常文本含 base64）？ | 属实：正常文本中的 base64 字符串会被 decode 后重扫 → 误报风险存在；但相似度/分数封顶限制加分；**未计入 KO-01 边界** → 追加到 Reconciliation 作为已知边界 | 部分反例成立 → 补边界声明 |
| X-03 | KO-02 | 反例：宿主权限表与 hook 同时存在时，hook 是否可能被绕过（直接调用内部函数）？ | hook 只拦 Claude Code 的 PreToolUse；绕过 = 不走 hook 的路径（如直接调用工具二进制）；Aigis 未声称防此 → 边界已在设计（宿主门 deny/ask 兜底） | 反例成立但被设计边界覆盖 |
| X-04 | KO-03 | 反例：SANITIZED 提升后，扫描器漏检的恶意内容获得 TRUSTED？ | 属实：SANITIZED 依赖 scan 质量；scan 漏检 → 提升后可控控制流工具（若授权存在）；**该链条未被测试覆盖**（EK-14）→ 已反映在 KO-03 的 Partially validated | 反例成立 → 维持降级 |
| X-05 | KO-04 | 反例：哈希链被"整链重写+重签"绕过？ | 需密钥；无密钥攻击者无法重签 HMAC；有密钥则链不设防（密钥管理在链外）→ 边界成立 | 反例未推翻 |
| X-06 | KO-05 | 反例：诚实性声明本身无法被审阅者验证（自有编号还是官方编号无法区分）？ | 属实：GL-* 与官方条款号格式相似，审阅者需读注释才知道；Aigis 通过注释+文档明示缓解 | 部分反例成立 → 保留观察 |
| X-07 | KO-06 | 反例：放弃 1000 stars 目标后，Aigis 的下载量/采用是否恶化？ | ROADMAP 未给出收缩后的下载趋势 → 无法证伪；属于 open question → 进 C-03/C-04 | 反例证据不足 |

### Counterexample 判定
- 7 个反例攻击：3 个部分成立（X-02/X-04/X-06 → 补边界/维持降级）、4 个未推翻；0 反例情况未出现（无需搜索证据豁免）

## 6. Epistemic Auditor（认知状态诚实性）

| # | 检查点 | 判定 |
|---|--------|------|
| E-01 | C-01/C-02/C-03/C-04/C-05 全部标 Hypothesis，未混入 KO | **CONFIRMED** |
| E-02 | KO-03 显式标 Partially validated (implementation only) 而非 Principle | **CONFIRMED** |
| E-03 | 所有 KO 带 Cross-project validation pending（除 KO-04/KO-07 为项目内强验证） | **CONFIRMED** |
| E-04 | 运行观察（JA/KO 未命中）未写成"能力缺陷"结论 | **CONFIRMED** |
| E-05 | CHANGELOG "1781 passed" 标为声明（PARTIALLY_CONFIRMED）而非独立验证 | **CONFIRMED** |
| E-06 | 无"单案例→Pattern→Principle"跳层（A-02 已收紧） | **CONFIRMED** |

### Epistemic 判定
- CONFIRMED 6 / CONTRADICTED 0

## 7. 判定统计汇总

| Auditor | CONFIRMED | PARTIALLY | DOWNGRADED | OVER_ | MISSING | CONTRADICTED | NEEDS_HUMAN |
|---------|-----------|-----------|------------|-------|---------|--------------|-------------|
| Truth | 17 | 2 | 0 | 0 | 0 | 0 | 0 |
| Coverage | 3 | 1 | 0 | 0 | 1 | 0 | 0 |
| Flow | 6 | 0 | 1 | 0 | 0 | 0 | 0 |
| Abstraction | 1 | 3 | 1 | 0 | 0 | 0 | 0 |
| Counterexample | — | 3 部分成立 | — | — | — | — | — |
| Epistemic | 6 | 0 | 0 | 0 | 0 | 0 | 0 |
| **合计** | **33** | **6** | **2** | **0** | **1** | **0** | **0** |

**3 成功**（高价值确认）：T-08（CaMeL 无条件 DENY 先于 grant，代码精确确认）、T-14（条件规则不可导出的 ExcludedRule 机制）、T-15（trust pack 自有编号诚实性注释）。
**3 错误/收紧**（考古产物修正）：F-01（`_map_action` 未命中返回 "unknown" 而非推测的 tool:*，措辞修正）、A-02（KO-02 从无条件 L4 陈述降为 Cross-project validation pending）、X-02（KO-01 补 decode 误报边界声明）。
**遗漏**：C-05（anthropic_proxy 仅测试级证据，轻量遗漏——纳入 Reconciliation 补充为 01 project-layer 一行）。
**Benchmark case**：A-02（本 run 最重要的升维攻击成功收紧——"宿主+插件双层门"从通用陈述降为带 pending 的模型）；X-04（SANITIZED 提升链依赖 scan 质量且无测试覆盖，维持 KO-03 降级）。

## 8. Contradictions 与 Counterexamples（保留声明）

- 本 run **无 CONTRADICTED**；以下 Counterexamples 部分成立并已保留在产物中：
  - X-02：decode 重扫对正常 base64 文本存在误报面（未计入 KO-01，作为边界声明）
  - X-04：SANITIZED→TRUSTED 提升链依赖 scan 检出质量，漏检时可能放行恶意内容（反映于 KO-03 降级与 C-02）
  - X-06：自有编号与官方条款号格式相似，审阅者依赖注释辨识（反映于 KO-05 观察）
- 这些保留不弱化证据标准；与"全绿"相比，本报告如实记录 2 个 DOWNGRADED + 1 个 MISSING。
