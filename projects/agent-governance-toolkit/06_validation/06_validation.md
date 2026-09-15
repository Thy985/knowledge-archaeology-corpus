# 06 — Validation & Evidence

## 1. 证据清单（Evidence Map）

| ID | 声明 | 证据 | 强度 |
|---|---|---|---|
| E-01 | govern() 两行集成 | govern.py:701 + README | S4（测试通过） |
| E-02 | 执行顺序 ring→policy→approval→audit | govern.py:231-290 代码顺序 | S4 |
| E-03 | fail-closed 无开关 | ADR-0013 + policy.py:168-172 | S4 |
| E-04 | 四种冲突策略 | policy.py:704-725 + 测试 | S4（134 passed） |
| E-05 | Merkle 审计链 | audit.py:268 + ADR-0017 + 测试 | S4 |
| E-06 | Decision BOM 可重构 | decision_bom.py + ADR-0018 + 测试 | S4 |
| E-07 | trust 四维加权 0-1000 | trust-score-calibration.md | S3（文档实现） |
| E-08 | trust ceiling 单调收敛 | ADR-0016 + identity/delegation.py | S3 |
| E-09 | ACS stateless/deterministic/fail-closed | policy-engine/README.md + docs/security-model.md | S3 |
| E-10 | ACS core 是 deprecation shim | policy-engine/core/src/lib.rs:1-56 | S3 |
| E-11 | 四执行环取代 RBAC | ADR-0002 + enforcer.py + 64 passed | S4 |
| E-12 | MCP gateway 治理 | mcp_gateway.py:1-35 | S3 |
| E-13 | Rego 输入形状坑 | govern.py:720-760 | S3 |
| E-14 | extends additive-only | policy.py:750-770 + ADR-0014 | S3 |
| E-15 | 本地测试 368 passed | 本轮 pytest 实测 | S4 |
| E-16 | 4 失败 = 缺 agentrust-trace | trace_sink.py:185-187 | S4（环境性） |
| E-17 | 13,834 test fn | 全仓 grep | S3 |
| E-18 | 992 conformance / 29 ADR | README + docs/adr/ | S7（项目自述） |
| E-19 | 威胁模型 7 类 | SECURITY.md | S3 |
| E-20 | PR #2794 CI 退化史 | pyproject 注释 | S3 |

## 2. Validators 摘要

### Truth Auditor（盲重建）
独立重读 govern.py / policy.py / audit.py / decision_bom.py / enforcer.py / SECURITY.md / ADR-0013/16/17/18/30/32 后对比：
- **无事实错误**：全部 EK 的 file:line 引用经独立重读确认存在且语义一致
- **E-15/16** 为本地实测（非推断）
- **E-18**（992/29）标注 S7 自述，未当作 S4 本地验证——诚实分级

### Coverage Auditor（独立重搜）
- 强制子系统覆盖：policy/identity/audit/trust/approval/rings/mcp/ACS/compliance/benchmarks 均入 EK
- **已知缺口**：9 语言 SDK 细节、agent-sre 全貌、agent-lightning 训练治理、docs/compliance 全量映射、redteam benchmark 完整跑分——未覆盖，标于 Snapshot 边界
- **遗漏风险**：agent-marketplace（插件签名）仅一句带过；compliance 框架（EU AI Act 映射）未深入——本轮聚焦治理核心机制，按项目规模缩放

### Flow Auditor（Flow Edge 真实性）
- Control/State/Data/Authority/Memory/Policy/Evidence 七类流全部回溯到 symbol + file:line
- 无"架构想象"Edge——每个 Edge 有代码位置

### Abstraction Auditor（升维判定）
- KO-01~08 全部经 Abstraction Promotion Gate：每级论证解释范围扩大
- 反例预算：KO-02 反例=AI Protector ML-in-path；KO-03 反例=C-04 合法跳跃场景；KO-05 反例=Guardian 审计链单层；KO-06 反例=Guardian 单包无 OS 移植
- 无"单项目→普适定律"的越级升维；跨项目 KO 标 Cross-project validation pending

### Counterexample Hunter（反例预算）
- 每 KO ≥2 定向反例攻击（见上），0 反例 KO 无（全部有对照项目或通例）
- bypass/alternate 路径专项：
  - `rego_path` 逃生舱 → EK-22/22b（静默不匹配坑已记录）
  - ACS URL extends（manifest provenance 门控缺失）→ EK-17
  - deprecation shim → EK-16（决策逻辑外移 crates.io）
  - `on_deny` 回调可吞掉 deny（返回而非 raise）→ 记录为设计面（审批/deny 定制路径）
  - MCP gateway 是独立进程还是库内？→ 未核实，标 Candidate 之外（实现细节待确认）

### Epistemic Auditor（认知状态）
- Fact（E-01~20）→ Observation（测试行为）→ Hypothesis（C-01~10）→ Validated Pattern（KO-01/04）→ Principle（KO-02/03/05）→ Law（无，不升）
- 无 Hypothesis 冒充 Fact；无 Cross-project Candidate 冒充已验证 Principle

## 3. Quality Metrics（run_metadata.yaml 同步）

| 指标 | 值 |
|---|---|
| EK 数 | 36（+1 子条目 22b） |
| KO 数 | 8（L3×3 / L4×4 / L5×1） |
| Candidates | 10（Hypothesis×6 / Cross-project×4） |
| Facts/Evidence | 20 条证据清单 |
| EK 平均出边 | >2（全部声明 links） |
| 游离 EK | 0（36/36 有边） |
| KO 聚合规则覆盖 | 100%（R1-R4 均用） |
| Flow 七类 | 全覆盖（Control/State/Data/Evidence/Authority/Memory/Policy） |
| 本地测试 | 368 passed / 15 skipped / 4 env-failed |
| 测试失败归因 | 100% 归因（缺 agentrust-trace 可选依赖） |
| 跨项目对照 | Aigis / Guardian / AI Protector（四选实测第 4 点） |

## 4. Reconciliation（设计意图 ≠ 实现 ≠ 测试 ≠ 运行时）

| 项 | design_intent | implementation | tested | runtime |
|---|---|---|---|---|
| fail-closed | ADR-0013 明确无开关 | policy.py:168-172（allow 规则失败→fail-open 警告，其余 deny） | 测试覆盖 deny 路径 | 本地 368 通过 |
| trust ceiling | ADR-0016 | identity/delegation.py + env 变量 | 未单测（本轮） | 未运行 |
| ACS 确定性 | spec 声明 | core 是 shim，真实在 crates.io | conformance（未本地全跑） | 未运行 |
| TRACE | ADR-0032 | trace_sink.py | 4 失败（缺依赖） | 未运行 |
| Decision BOM | ADR-0018 可重构 | decision_bom.py | 134 passed 含 BOM 测试 | 未运行 |

**Contradictions / Counterexamples 保留**：
- C-04：trust ceiling 可能误伤合法委托（ADR-0016 自承 tradeoff）
- C-09：deprecation shim = 仓库快照不证明决策逻辑
- E-17 vs E-18：13,834 test fn vs 992 conformance 声称的缺口未精确归因
- KO-02 反例：AI Protector 证明"ML 在路径内"也可工程化，确定性非唯一可行

## 5. Benchmark / Regression 候选（供 skill CI 使用）

- **B-01**：prompt-injection smoke corpus（280 行，110 attack/170 benign）——可作跨项目基准输入（AGT/AI Protector/Guardian 同一 corpus 对比）
- **B-02**：fail-closed 断言测试（策略异常 → deny 而非 allow）——AGT 已有测试，可抽为 skill 的 agent-governance 品类 regression case
- **B-03**：trust ceiling 单调性测试（父 300 → 子 clamp ≤300）——可抽为 regression
- **B-04**：Decision BOM 重构一致性测试（事后重建 == 决策时事实）——可抽为 regression
- **B-05**（Reconciliation 新增）：advisory 层"只能收紧不能放宽"契约测试——block 后不允许 allow 覆盖；失败默认 allow 且审计 deterministic:false
- **B-06**（Reconciliation 新增）：authority resolver 兜底路径测试——无 YAML 规则匹配时 trust narrowing 生效
- **B-07**（Reconciliation 新增）：SSRF 防护测试——cloud metadata IP 域名/URL 拒绝（_BLOCKED_HOSTS）

## 6. Independent Validation 摘要（详细见独立报告）
- 判定：21 CONFIRMED / 3 PARTIALLY_CONFIRMED / 1 DOWNGRADED / 1 OVER_GENERALIZED / 2 MISSING / 1 CONTRADICTED / 2 NEEDS_HUMAN_REVIEW
- 3 成功：决策管线顺序确认 / fail-closed 无开关 / 审计四层证据机制
- 3 错误：advisory 层遗漏 / authority resolver 遗漏 / Merkle 链+树表述不准
- 修正全部并入本 Package（EK-37/38、EK-08/33、KO-02/06、Flow Atlas）
