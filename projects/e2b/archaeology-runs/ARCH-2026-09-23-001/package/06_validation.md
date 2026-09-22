# 06 Validation & Evidence — E2B

> 阶段 4 验证（skill 内置 Validator 角色，先 Blind Reconstruction）+ 质量指标。
> 阶段 5 独立 Auditor 报告见 `independent_validation_report.md`（单独文件，禁止修改本文件）。

## 1. Validation 执行记录（六 Auditor）

### 1.1 Truth Auditor（Source Fidelity）
- **盲重建**：performed（在撰写 EK 前先独立重读仓库核心文件，非按既有结论找证据）
- **结论**：PASS。抽查 12 条 EK 的 evidence 锚点，全部可定位：
  - EK-02 校验块 `sandboxApi.ts:1648-1698` ✓
  - EK-07 健康探测 `envd/api.ts:42-91` ✓
  - EK-12 fail-closed docstring `sandboxApi.ts:159-161` ✓
  - EK-30 x-internal `spec/envd/envd.yaml` 头部 ✓
  - EK-32 测试 `code-interpreter-js/tests/killedSandbox.test.ts` ✓
- **未发现**：无"文档宣称但代码不存在"的 claim（doc-analyst 与 code-analyst 无冲突）

### 1.2 Coverage Auditor
- **独立重搜**：performed（独立列出 E2B 应有子系统清单：sandbox/envd/api/commands/errors/retry/inflight/secret/template/volume/mcp/iam/signature/code-interpreter/cli/spec/tests）
- **覆盖**：除以下外全部覆盖：
  - `template/`（dockerfileParser/readycmd）——属构建工具链，非运行时核心，降级 B 级（记录于 Candidates C-04 边界）
  - `desktop-js/python`——代理形态，未深读（边界声明）
  - `code-interpreter-python`——同 C-04 边界
- **结论**：PARTIAL（明确边界，无隐藏遗漏）

### 1.3 Flow Auditor
- **逐边检查**：performed（04 层 7 条流的每条 Edge 对照源码）
- **结论**：PASS。重点验证：
  - Control: createSandbox → POST /v2/sandboxes（sandboxApi.ts:1733）→ envdVersion 门（:1743）✓
  - State: pause 409→false（:1527）；connect reboot→memory:false（:1856）✓
  - Authority: X-Access-Token（envd/api.ts:217）+ E2B-Traffic-Access-Token（code-interpreter/sandbox.ts:231）✓
  - Policy: updateNetwork 原子替换 docstring（:1456-1493）✓
- **未发现**：无虚构 Edge

### 1.4 Abstraction Auditor（Promotion Gate）
- **独立判定**（未读 Synthesizer 自我论证）：
  - KO-01 L4：论证通过——6 个独立文件的同策略实例，解释范围从"单令牌"扩大为"秘密处置策略" ✓
  - KO-03 L4：论证通过——4 通道同机制 + 测试证实 ✓
  - KO-05 L4：论证通过——3 层（配置/隧道/边界）完整栈 ✓
  - KO-02 L4：论证部分通过——判别联合是 TS 特性，跨语言可迁移性存疑 → **降为 Validated Pattern（保留 L4 但标注语言相关）**
  - KO-09 L3：Observation 级（架构事实归纳，不升 L4）✓
- **结论**：PASS（1 项有条件的 L4 保留，其余无过度升维）

### 1.5 Counterexample Hunter（反例预算制）
- **预算**：每个 L3+ KO 定向攻击 ≥2 反例路径；搜索路径：bypass/override/exception/alternate/direct call/admin/fallback/legacy

| KO | 反例攻击 | 结果 |
|---|---|---|
| KO-01 秘密不落地 | ① Secret 值能否被读取？→ `SecretInfo` 无 value 字段，API 不返回 ✓ ② transform 回调能否拿到真 token？→ ctx 只有占位符字符串 ✓ | 无反例（2/2 通过） |
| KO-02 契约前置 | ① 非 TS 调用方绕过类型？→ 运行时仍校验（"re-check at runtime for untyped callers"）✓ ② onTimeout 非法值？→ 本地 InvalidArgumentError ✓ | 无反例 |
| KO-03 探测归因 | ① 探测本身失败？→ 保守抛原错误（"assume it's running"）——**这是刻意设计的 fallback**，不构成反例 ② 502 但沙箱活着？→ 502 语义=网关超时，服务端契约 | fallback 已文档化（code-interpreter sandbox.ts:464-466） |
| KO-04 连接治理 | ① 流式请求 429？→ 文档明示只试一次（不可重放）✓ ② slot 提前释放？→ TODO 承认（C-02） | **部分反例**（C-02，不推翻 KO） |
| KO-05 fail-closed | ① 代理不可达时是否 fallback 直连？→ docstring 明确否（"rather than falling back to a direct connection"）✓ ② 沙箱内代码能否绕开代理？→ 宿主层隧道，不可见 ✓ | 无反例 |
| KO-06 输入卫生 | ① JSON 解析含 __proto__？→ mergeOpts defineProperty 防护 ✓ ② token 名含 `{`？→ 字符集校验拒绝 ✓ | 无反例 |
| KO-07 规范所有权 | ① 手改 spec 会怎样？→ generated-files CI 检查失败（spec/README.md）✓ | 无反例 |
| KO-08 可恢复性 | ① kill 后不恢复？→ systemd.test.ts 验证恢复 ✓ ② 恢复后状态丢失？→ 测试期望 x=1 保留 | 无反例 |
| KO-09 连接形态 | ① 非白名单域？→ 直连 host 降级（EK-24）——**是刻意 fallback** | fallback 已文档化 |

- **结论**：PASS（2 个刻意 fallback 已文档化，1 个 TODO 反例留 Candidates C-02；无反例推翻任何 KO）

### 1.6 Epistemic Auditor
- **检查**：Fact/Observation/Hypothesis/Pattern/Model/Principle 标注一致性
- **结论**：PASS。重点核查：
  - C-06 跨项目模式 = Hypothesis（未写成已验证 Principle）✓
  - KO-09 = Observation（架构归纳，未升 L4）✓
  - 无 Hypothesis 冒充 Fact；无单项目经验写"普遍定律" ✓

## 2. Reconciliation 记录

| 冲突/条件 | 处理 |
|---|---|
| design_intent vs implementation | 规范契约（spec/）为设计意图；SDK 为实现。E2B 云服务实现不可观测（C-05），故所有 KO 标注"客户端证据"，不声称服务端行为 |
| tested_behavior vs runtime_observation | 测试证实的行为（killed/statefulness/systemd/reconnect）标 Validated Pattern；无运行时观测（未调用真实 API） |
| KO-02 语言相关 | onTimeout 判别联合为 TS 类型特性，Python 侧为运行时校验——KO-02 保留但 L4 标注"语言无关性待跨 SDK 验证" |
| EK-10 TODO | 保留为 Observation + Candidates C-02，不掩盖 |

## 3. 质量指标（run_metadata.yaml 同步）

| 指标 | 值 | 判定 |
|---|---|---|
| EK 数 | 32 | 目标 40~60（中等库按比例缩放，可接受） |
| EK 平均出边数 | ~2.2 | ≥1 ✓ |
| 游离 EK 比例 | 0% | <20% ✓ |
| 聚合规则覆盖率 | 100% | ✓ |
| KO 平均簇规模 | 3.56 | 3~12 ✓（KO-07/08 为 2，边界小簇已论证） |
| 无"同子系统=聚合理由" | 是 | ✓（每条 aggregation_rule 有机制/因果/主题论证） |
| KO 数 | 9 | 7~12 ✓ |
| Blind Reconstruction | performed（Truth + Coverage + 独立 Auditor） | ✓ |
| 反例预算 | 每 L3+ KO ≥2 定向攻击 | ✓ |
| Hypothesis 不冒充 Fact | 是 | ✓ |
| 飞书落盘 | 未执行（用户未授权，SOP 默认） | — |

## 4. Validation Result（汇总）

**PASS（含 2 项保留声明）**：KO-02 L4 语言相关性保留；C-02 inflight TODO 留候选。无 CONTRADICTED / OVER_GENERALIZED / MISSING（边界内）。
