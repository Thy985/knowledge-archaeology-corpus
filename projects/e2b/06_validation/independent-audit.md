# Independent Validation Report — ARCH-2026-09-23-001 (E2B)

> 阶段 5 产物。独立 Auditor 未把 Archaeology Result 当事实来源：先独立重读仓库关键文件（connectionConfig.ts 615 行、envd/api.ts 231 行、errors.ts 256 行、sandbox/sandboxApi.ts 关键段落、sandbox/iam.ts 149 行、code-interpreter-js/src/sandbox.ts、sandbox/commands/index.ts 489 行），建立 Independent Findings 后再与考古结果逐条对比。
> **本报告不修改任何原考古产物。**

## 1. Independent Findings（盲重建摘要，与考古结果无关）

Auditor 独立从代码读到的关键事实：
1. **iam.ts Proxy 语义**：`get` trap 命中 RUNTIME_PROBED_PROPS（toJSON/then/toString/valueOf）时**返回 stand-in 值（runtimeProbeValue），不抛**；真正"使用"（Symbol.toPrimitive / toJSON 调用）时 resolve 才抛 InvalidArgumentError。docstring 明言"Reading one cannot throw — otherwise JSON.stringify(iam.tokens) inside a callback would"。
2. **createSandbox 强校验**：三处 InvalidArgumentError（keepMemory 仅 pause / autoResume 仅 pause / !keepMemory×autoResume）全部在 body 构造之前；onTimeout 未配置时 **autoPause 字段整个省略**（`onTimeoutConfigured ? action==='pause' : undefined`）——"leaves the timeout action to the API"。
3. **envdVersion 门**：compareVersions < 0.1.0 → `await this.kill(sandboxID)` + TemplateError（补偿动作，防沙箱泄漏）。
4. **健康探测**：checkSandboxHealth 5s 超时、502→false、其他非 ok→undefined；handleEnvdApiFetchError 仅当 `running === false` 才归因 TimeoutError（"killed or reached its end of life"），undefined/true 抛原错误。
5. **两阶段超时**：setupRequestController 握手超时 → clearStartTimeout → createIdleAbort（idle-read，只限网络侧，arm per chunk）；abortWithReason pin reason（Bun 1.3.14 weak 引用 GC 问题）。
6. **ServiceBusyError**：`extends Error`（不在 SandboxError 树），statusCode=503，"Nothing was changed by the refused call"，docstring 要求显式 catch。
7. **fail-closed**：egressProxy docstring——宿主层隧道、过滤之后、UDP（DNS/QUIC）不隧道、不可达时"fail rather than falling back"、DNS dial 时重解析并 pin、私网地址沙箱创建前拒绝。
8. **updateNetwork**：PUT /sandboxes/{sandboxID}/network，docstring"Replaces ... atomically — fields that are omitted are cleared on the server"。
9. **runCode**：requestTimeout 握手（默认 30s）+ bodyTimeout 执行（默认 60s）+ keepalive:true + context_id；handleRequestError 中 `(await this.isRunning().catch(() => true)) === false` → 描述性 TimeoutError（探测失败保守假设在跑）。
10. **commands**：模块在 `packages/js-sdk/src/sandbox/commands/index.ts`（489 行）——/bin/bash、SIGKILL、supportsStdinClose、KEEPALIVE_PING_HEADER。

## 2. 对比判定统计

| 判定 | 数量 | 涉及 |
|---|---|---|
| CONFIRMED | 20 | EK-01/02(主体)/03/05/07/08/09/10(主体)/11(主体)/12/13/15/17/20/21/22/26/27/28/32 |
| PARTIALLY_CONFIRMED | 4 | EK-11(省略语义补充)/EK-14(机制描述不完整)/EK-16(表述可精确化)/EK-19(证据路径修正) |
| DOWNGRADED | 0 | — |
| OVER_GENERALIZED | 1 | KO-02 L4 语言相关性（06 层已自行降级保留，Auditor 复确认必要） |
| MISSING | 3 | iam `has` trap；JWT-SVID 类型；onTimeout 省略→C-03 强相关 |
| CONTRADICTED | 1 | **EK-14"读取即抛"与实现相反** |
| NEEDS_HUMAN_REVIEW | 0 | — |

## 3. 原 Archaeology 最重要的 3 个成功

1. **EK-02/EK-03 生命周期契约前置 + 版本门**：逐行核实，与实现完全一致。含"untyped callers runtime re-check"、`autoPause` 条件省略、kill 补偿动作——这是 E2B 工程质量的真实骨架，考古准确捕获。
2. **EK-07/EK-28 保守降级错误归因**：跨 envd REST/gRPC/code-interpreter 三实例独立验证，`running===false` 才断言"沙箱被杀"、探测失败抛原错——该 Pattern 提炼（KO-03）站得住。
3. **EK-12 fail-closed 网络面**：docstring 与 Flow Atlas 断言完全一致（宿主层隧道/过滤后/fail-closed/DNS pin/私网拒绝/UDP 不隧道），无虚构，且正确识别出"与 allowOut/denyOut 过滤的顺序关系"这一关键不变量。

## 4. 原 Archaeology 最重要的 3 个错误

1. **【CONTRADICTED】EK-14 claim 与实现相反**：考古结果写"RUNTIME_PROBED_PROPS 在**读取即抛**"；实现是读取时返回 stand-in、**使用即抛**（iam.ts:118-146）。设计意图恰恰是"读取不能抛，否则回调里 JSON.stringify(iam.tokens) 会崩溃"（iam.ts:17-23）。这是行为方向性错误，会影响读者对该防护机制的理解。
2. **【PARTIALLY_CONFIRMED】EK-19 证据路径错误**：写 `packages/js-sdk/src/commands/index.ts`，实际在 `packages/js-sdk/src/sandbox/commands/index.ts`（subsystem 归属应为 sandbox 内）。证据可追溯性受损——若未来有人按路径查证会扑空。
3. **【OVER_GENERALIZED 风险】KO-02 L4"客户端契约前置"**：判别联合（discriminated union）是 TypeScript 静态类型特性，Python SDK 侧无此静态保证（运行时校验）。06 层已自我降级标注，Auditor 确认该降级必要且应保持——若不降级，是典型的"语言特性被升维为通用模式"。

## 5. 是否存在关键遗漏？—— 是（3 项，均不推翻 KO）

- **MISSING-1**：iam.ts `has` trap（`name in iam.tokens` 只认自有键，不报 Object.prototype 继承成员）——与 `__proto__`/`constructor` 防护同一机制，EK-14 漏了。
- **MISSING-2**：`SandboxIamTokenType = 'JWT-SVID'`（服务端定义类型集，"may grow, so any string is allowed"）——EK-13 附近可补。
- **MISSING-3**：onTimeout 未配置 → autoPause 字段整体省略（API 拥有默认值）——与 C-03 强相关，考古已把该语义放 Candidates，但 EK-02 正文未显式记录"省略即让权给 API"。

## 6. 是否存在错误升维？—— 是（1 项，已被 06 层自行降级）

KO-02 L4 的语言相关性（见 §4-3）。其余 KO 升维（KO-01/03/05 L4）经 Auditor 独立论证，均站得住。

## 7. 是否存在事实错误？—— 是（1 项）

EK-14 的"读取即抛"（§4-1）。其余抽查的 20 条 CONFIRMED 无事实错误。

## 8. 是否存在 Flow 错误？—— 否（但 1 处路径需补全）

七条流抽查通过：Control（校验→POST→版本门）、State（pause 409→false @ sandboxApi.ts:1527）、Authority（X-Access-Token @ envd/api.ts:216、E2B-Traffic-Access-Token @ code-interpreter/sandbox.ts:231、KEEPALIVE_PING_HEADER @ commands/index.ts:347,461）、Policy（updateNetwork 原子替换）均实证。
唯一修正：Flow Atlas 中 commands 引用路径补全为 `packages/js-sdk/src/sandbox/commands/index.ts`。

## 9. 是否发现新的 Benchmark / Regression Case？—— 是（2 个）

1. **【Benchmark】IAM Proxy 读取 vs 使用语义**：判断一个知识对象是否正确描述了"读取给 stand-in、使用即抛"——可用于未来所有含 Proxy/占位符系统的考古基准（若考古把读取即抛写成使用即抛，即 FAIL）。
2. **【Regression】EK 证据路径存在性检查**：EK-19 路径错误本可被"evidence 路径必须真实存在"的自动检查捕获。建议作为 skill 级回归：任何 EK 的 evidence symbol 路径必须在仓库中可 grep 到。

## 10. 结论

原 Archaeology 总体质量高（20 CONFIRMED / 1 CONTRADICTED），核心机制、Flow、KO 提炼均扎实。**1 处行为方向性事实错误（EK-14）+ 1 处证据路径错误（EK-19）需在 Reconciliation 中修正**；3 项遗漏补入 EK 正文。无推翻性反例，无 Benchmark 级失败。详细 Reconciliation 见阶段 6 corpus 提交。
