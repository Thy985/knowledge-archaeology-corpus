# Independent Validation Report — ARCH-2026-09-15-001

> Auditor 盲重建：不把原 Archaeology Result 当事实来源。独立重读仓库（HEAD 8f57f85）建立 Independent Findings 后与 Package 对比。**未修改原考古产物**。

## 判定统计

| 判定 | 数量 | 明细 |
|---|---|---|
| CONFIRMED | 8 | graph 三态路由 / 加权风险分 / NeMo 软贡献注释 / A2 反混淆(raw 首位) / output_filter 三线 / RBAC 模型 / ISS-003/004 幻影 / ISS-012 6 xfail |
| PARTIALLY_CONFIRMED | 2 | 基准数字（S7 自述非独立复现）/ KO-08"输出侧最完整"（需与 Aigis exfil 对照） |
| DOWNGRADED | 0 | — |
| OVER_GENERALIZED | 0 | KO-02 三态已正确标 2 项目；KO-01/09 数对 3 项目 |
| MISSING | 3 | direct endpoint（默认开启）/ streaming 无输出过滤 / 管理端点无认证 |
| CONTRADICTED | 0 | — |
| NEEDS_HUMAN_REVIEW | 2 | direct endpoint 生产默认开启 + 管理面无认证（部署安全性） |

## 一、原 Archaeology 最重要的 3 个成功

1. **"provable"闭环被正确识别为品类内独有能力**（KO-06 + EK-25/26/27）——Benchmark Hub + 客观真值 grader（94%→99%，no LLM-as-judge）确实是 Aigis/Guardian 都没有的架构层承诺，且与雷达 Proof-of-Guardrail 主题建立了可验证连接（C-01）。
2. **A2 反混淆作为独立机制被完整提炼**（EK-06/07/08/09/12 + KO-04），包括 variants-first 实验证据（-13pp）与 fail-safe 设计——这是 Guardian（regex 静态匹配）缺失的检测前置层，对照价值高。
3. **失败与修复史诚实保留**（EK-05 NeMo 软贡献、EK-20/21 热更新+预加载、EK-24 依赖低估修复、EK-22 幻影功能）——"检测层错不崩热路径"（KO-03）有真实的认知成本支撑，非推断。

## 二、最重要的 3 个错误

1. **遗漏：direct bypass endpoint 未进 Engineering Knowledge**——`routers/direct.py:43 POST /v1/chat/direct` 显式绕过（docstring："NO scanning, NO policy, NO audit log. For Compare demo only"），且 `config.py:117 enable_direct_endpoint: bool = True # Set False in production` **默认开启**。原 Package 的 7 类 Flow 与 EK 均未记录该 alternate path。**影响**：绕过路径是安全网关最重要的攻击面之一，缺失会导致未来复用者高估防护覆盖。
2. **遗漏：streaming 路径无输出过滤**——`routers/chat.py:129-164`：stream 请求只跑 `run_pre_llm_pipeline`（到 decision），ALLOW/MODIFY 后直接 `sse_stream`；`llm/streaming.py` 无 output_filter/PII/secret/system-leak 引用。**输出侧三防线（EK-14）只覆盖非流式响应**——原 Package 未限定范围，属于覆盖过度声明。
3. **遗漏：管理端点无认证**——`main.py` 仅 CORSMiddleware + CorrelationIdMiddleware；`routers/policies.py / rules.py / analytics.py / requests.py` 的 CRUD 端点 Depends 只有 get_db（无 auth）。任何可达 :8000 者可改策略/规则、查请求日志。原 Package 的 Authority Flow 未覆盖管理面授权边界。

## 三、是否存在关键遗漏

**是（3 项，见上）**。均为"绕过/管理面"类覆盖缺口，属于 Auditor 重点攻击类别（bypass / alternate path / admin path / fallback）。

## 四、是否存在错误升维

- 未发现错误升维。KO 分层保守：三态分级（KO-02）明确"2 项目 corroborated"；KO-06/KO-10 标 pending。
- 一处**边缘风险**：KO-08"AI Protector 是品类内输出侧最完整的"——Aigis（09-13）有 exfil 检测（输出方向），"最完整"表述需与 Aigis 的 exfil 实现对照后才能成立 → 判定 PARTIALLY_CONFIRMED，reconciliation 收紧措辞为"输出侧三线（PII/secrets/system-leak）为 AI Protector 独有；与 Aigis exfil 的对照待定"。

## 五、是否存在事实错误

- 无事实错误。抽查的 10 个核心 Claim 全部代码级确认（file:line 精确）。
- 一处**自述边界已正确标注**：基准数字（99%/91%/48ms/1900+）为 S7，本地实测 668 passed 与项目自述差额已如实记录（C-07）。

## 六、是否存在 Flow 错误

- 无 Flow 错误。graph 三态路由、MODIFY 路径（transform→llm_call）、pre_llm_only、BLOCK 短路均独立确认。
- **Flow 覆盖缺口**：Flow Atlas 未含 direct endpoint 旁路（bypass flow）与 streaming 输出旁路——reconciliation 需补充。

## 七、是否发现新的 Benchmark / Regression Case

**是（2 项候选）**：
1. **R-AP-001：direct endpoint 门控回归测试**——断言 `enable_direct_endpoint=False` 时 `/v1/chat/direct` 返回 403，且默认值（demo）不影响。基准：确保 bypass 端点默认关闭或显式告警。
2. **R-AP-002：streaming 输出过滤覆盖测试**——构造含 PII/system-leak 的流式响应，断言 sse_stream 不泄露（当前实现预期 FAIL → 记录为已知缺口回归基线）。

## 八、Epistemic 状态检查

- Fact/Observation/Hypothesis/Pattern/Model/Principle 无混淆 ✓
- "No LLM in the loop" 语义歧义已入 Candidates（C-03）✓
- 项目自述（S7）与本地实测（S4）分离 ✓

## 结论

原 Archaeology 核心机制层可信（10 核心 Claim 全 CONFIRMED），升维纪律良好（0 OVER_GENERALIZED / 0 CONTRADICTED），但**覆盖存在 3 处关键遗漏，均集中在"绕过/管理面"攻击面**，且其中 2 项触及生产部署安全性（NEEDS_HUMAN_REVIEW）。Reconciliation 需：①补 3 条 EK（direct/streaming/admin-auth）；②收紧 KO-08 措辞；③Flow Atlas 补 bypass flow。
