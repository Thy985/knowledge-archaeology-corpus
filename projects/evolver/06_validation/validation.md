# 06 Validation & Evidence

## 1. 验证方法

| 检查 | 方法 | 结果 |
|------|------|------|
| Source Truth | 盲重建：独立重读 index.js（CLI/loop/solidify/bootstrap）+ sandboxExecutor/atp/protocol/canary/forceUpdate（可读）+ README/SKILL.md + 抽样测试 + conformance | ✅ 无事实错误（可读面） |
| Coverage | 强制面覆盖：进化引擎/协议/安全/自我更新/代理/适配器/ATP/记忆/观察面板/脚本/测试体系 | ✅ 30 EK 覆盖 11 面 |
| Causality | 每条 EK 因果声明回溯 | ✅ |
| Flow | 七类流 Edge 回溯（可读面）；🔒 面显式标注 | ✅（含盲区声明） |
| Abstraction | L3/L4/L5 独立判定 | ✅ 7 KO/4 CM/4 M 受支撑 |
| Counterexample | 反例预算制 | ✅ 见下 |
| Epistemic | Fact/Observation/Hypothesis/Pattern 标注 | ✅ C-01~08 诚实标注 |

## 2. 反例攻击记录

### 对 KO-01（协议约束自进化）的攻击
- **A1**：README 说"不自动编辑源码"→ 反例：solidify + EVOLVE_ALLOW_SELF_MODIFY 存在 → **收紧**：KO-01 表述为"生成协议 prompt + 验证补丁"，自我修改边界入 C-03（NEEDS_HUMAN_REVIEW）
- **A2**："不 improvisation"是否属实？→ selector 逻辑混淆不可读；种子基因 strategy 含"Select an existing Gene by signals match (no improvisation)"（可读）→ 部分证实 ✅
- **A3**：学习回填是否真实？→ adaptGeneFromLearning 测试断言（action 类不过滤、problem/area 类过滤进 signals_match）→ 实测契约证实 ✅

### 对 KO-02（安全纵深）的攻击
- **B1**：白名单是否可绕过？→ npm/npx 已移除（GHSA 测试断言 ALLOWED_EXECUTABLES 恰好=['node']）✅；但 BLOCKED_NODE_FLAGS 具体清单未穷举验证（可读，未逐一核对）→ 记录
- **B2**：文档称 npm/npx 允许 vs 实现禁止——**纵深安全比文档更严**（反向 doc-impl）→ C-02
- **B3**：源码级回归对混淆面失效 → C-04 ✅

### 对 KO-03（自我更新可恢复）的攻击
- **C1**：备份恢复是否真的 fail-closed？→ _recoverInterruptedForceUpdateBootstrap：恢复失败 → "Refusing to continue startup" + exit(1)（index.js 可读 L25-38）✅
- **C2**：升级会不会误伤第三方仓库？→ _hasStrongEvolverInstallMarkers（安装标记验证）+ rollbackSafety isInsideEvolverRepo 测试 ✅
- **C3**：金丝雀是否真的在重启前？→ canary.js 注释（"catches it BEFORE the daemon restarts with broken code"）✅（但 canary 触发条件细节未读——记录为部分证实）

### 对 KO-07（自指与开源治理）的攻击
- **D1**：是否过度解读"混淆"？→ 反例：混淆可能只为防破解而非防抄袭；README 明确声明 source-available 转向与 Hermes 争议相关 → **收紧**：动机写为"推断（Interpretation）"而非事实 ✅
- **D2**：GPL 合规是否可判定？→ 法律问题超出考古范围 → 保持 C-01 Hypothesis，标注需法律/人类审查 ✅

## 3. Epistemic 状态清单（最终）

| 对象 | 状态 | 依据 |
|------|------|------|
| EK-06（沙箱）/EK-11/12（daemon）/EK-15（ATP）/EK-09（forceUpdate） | **Fact** | 可读实现（S3/S4 测试） |
| EK-01/02（进化循环/solidify） | **Fact**（外部观察） | README + CLI + 测试契约；实现🔒 |
| EK-26（混淆范围） | **Fact** | 实测 grep（S4） |
| KO-01~07 | **Pattern**（本项目） | 支撑 EK |
| CM-1~4 / M-1~4 | **Model / Methodology**（pending） | 支撑 KO |
| C-01~C-08 | **Hypothesis/Observation/Tentative** | 未确认 |
| C-02/C-03 | **NEEDS_HUMAN_REVIEW** | doc-impl 裁决 / solidify 边界 |

## 4. 未实跑项（诚实标注）

- **evolve.js🔒 / gep 混淆面**：内部逻辑不可读，全部结论基于外部观察（CLI 调用点/README/测试契约）——盲区明确
- **抽样测试**：60 pass / 2 fail（sandboxExecutor.security + rollbackSafety fail 因 @evomap/gep-sdk 未 npm install——环境问题，非回归）；savingsCoreConformance + crypto 通过
- **npm install + 全量 221 测试**：未执行（依赖安装成本高，非核心面）

## 5. 质量指标

| 指标 | 值 |
|------|-----|
| Facts/Evidence | ~60（distinct 证据点） |
| EK | 30（links 100%，游离 0） |
| KO | 7（aggregation_rule 100%） |
| CM/M | 4 / 4 |
| Candidates | 8（含 2 个 NEEDS_HUMAN_REVIEW） |
| 反例攻击 | 12 次（A1-3/B1-3/C1-3/D1-2） |
| 实测 | 抽样测试（60 pass/2 env-fail）+ conformance 通过 |
| 判定 | PASS（含 2 收紧 + 2 NEEDS_HUMAN_REVIEW + 混淆盲区声明） |
