# 02 Engineering Knowledge — EK Graph（宽底座，L1/L2）

> 每条 EK 含 links（6 类边：mechanism/subsystem/causal/dependency/constraint/contrast）；evidence 可追溯符号/文件/commit。Engineering Knowledge 是独立价值的推理原材料，不因形成 KO 删除。

## A. 决策管线（write/read/delete）
### EK-01 默认 permissive：无参实例 = 检测+事件但不拦截
`MemoryGuard()` 无 policy 参数时用 `Policy.permissive()`（default_action=ALLOW）；只有 `Policy.strict()` 或 YAML 加载才启用主动执行。文档意图："intentionally permissive by default"。
- links: `contrast` EK-08（strict 预设）；`causal` → EK-02（permissive 下 write 仍走全管线）
- evidence: guard.py:52（`self._policy = policy or Policy.permissive()`）；policies/policy.py `permissive()`

### EK-02 write 五阶段管线：provenance → classification → detection → decision → commit
`MemoryGuard.write`：①coerce source_class（显式 source_class 优先，否则 source_type 兼容映射）→ ②分类检查（写会 reclassify 现有 key 则 BLOCK，须用 promote）→ ③注入 `_pending_source_class`（try/finally 复位）跑 `_run_detectors` → ④`_highest_severity` + `_decide`（BLOCK：emit+pre-block 快照+raise PolicyViolation；QUARANTINE：入 quarantine 字典不写 store；REDACT：改写值）→ ⑤`store.set` 提交，非 AGENT_AUTHORED 写记 `note_independent_write`，immutable 键自动 baseline。
- links: `causal` EK-05→EK-02→EK-07（检测→决策→提交）；`subsystem` EK-03（read 管线）；`dependency` EK-04（delete 是独立短路径）
- evidence: guard.py:250-470（write 全程）

### EK-03 read 双保险：完整性 verify 先行 + 出站泄漏筛查
`read`：先 `self.verify(key)`（IntegrityError → CRITICAL BLOCK 事件 + 抛出），再跑检测器 + `_decide`（BLOCK raise / REDACT 改写 / ALLOW-with-findings 事件）。
- links: `causal` EK-19→EK-03（integrity 基线支撑读校验）；`contrast` EK-02（写路径 vs 读路径）
- evidence: guard.py:470-540

### EK-04 delete 短路径：protected key 硬拦截
`delete` 直接查 `_protected_detector.matches(key)`，命中即 CRITICAL BLOCK + PolicyViolation，不做检测器全管线；否则 delete + integrity.clear + classification.clear + self_reinforcement.reset。
- links: `subsystem` EK-11（ProtectedKeyDetector）；`constraint` EK-11 约束 EK-04（保护名单决定可删集）
- evidence: guard.py:540-560

## B. 检测与不可检查内容
### EK-05 检测器异常 fail-open 但必发事件
`_run_detectors` 中 detector.inspect 抛异常 → verdict 丢弃（不能阻断 agent）→ 发 LOW 级 SecurityEvent（metadata detector_error=True）。"fail-open must not be silent"（#138）。
- links: `causal` → EK-22（事件承载）；`constraint` 约束 EK-02（异常不会静默放行）
- evidence: guard.py:620-645；commit 4c56524

### EK-06 _stringify 50 层截断：防 RecursionError 变静默绕过
`_stringify` 对嵌套容器 ≤50 层展开，更深替换为 `<amg:max-depth>`（TRUNCATION_MARKER）——检测器永不读到标记下内容；迭代式 `exceeds_max_depth` 自身不会触递归上限（#135）。
- links: `mechanism` EK-07（同一深度边界问题的两半）；`causal` → EK-07
- evidence: detectors/injection.py:80-140；commit c5904bf

### EK-07 超出检查深度 = size_anomaly finding，不是静默
#135 截断后，51+ 层载荷会被 ALLOW 且无事件（main@dde38e8 实测验证）；#151 修复：`exceeds_max_depth(value)` 为真 → 追加 size_anomaly HIGH finding → strict 隔离 / permissive 记录事件。"Content that could not be inspected is a finding, not silence."
- links: `causal` EK-06→EK-07；`constraint` EK-07 约束 EK-02（决策必须看见不可检查内容）
- evidence: guard.py:645-655；commit 24d1d26（含测试 test_uninspectable_nesting.py）

### EK-12 PromptInjectionDetector：11 条 regex，锚定防误报
ignore/disregard/forget previous instructions、jailbreak（you are now DAN...）、system: you are、XML 标签、act as admin、reveal/leak prompt、new instructions、override safety、`system override:` 锚定冒号/破折号（"thermostat's system override switch" 不命中）。
- links: `subsystem` EK-13（同为内容 regex 检测）；`contrast` EK-16（ML 检测器）
- evidence: detectors/injection.py DEFAULT_INJECTION_PATTERNS

### EK-13 SensitiveDataDetector：13 类 secrets/PII + redact
AWS AKIA/secret、ghp_/gho_、sk-、sk-ant-、AIza、xox、PEM、JWT、信用卡、SSN、email；`redact` 支持读/写净化。
- links: `mechanism` EK-12（regex 模式族）；`causal` → EK-02 的 REDACT 决策
- evidence: detectors/leakage.py DEFAULT_LEAKAGE_PATTERNS

### EK-14 SelfReinforcementDetector：冷却 + 自相似双规则
滚动窗口：60s 内 >3 次连续 agent_authored 写 → 标记；difflib 相似度 ≥0.85 的同键 agent_authored 写 = 强化前写而非独立佐证。防"幻觉/被建议内容经多次自我迭代固化为事实"（自中毒）。
- links: `subsystem` EK-15；`mechanism` EK-16（都依赖 provenance 语义）
- evidence: detectors/self_reinforcement.py（docstring "Layer 1 of the three-layer ASI06 defense"）

### EK-15 trust-aware decay：默认仅 SYSTEM 可衰减冷却
#124 修复：此前任何 external_tool/user_input 写都衰减计数（把不可信输入当独立证据）；现仅 `trusted_source_classes`（默认 {SYSTEM}）可衰减，USER_INPUT/EXTERNAL_TOOL/UNKNOWN 默认不可信。`note_independent_write` 只 popleft 一条（弱化而非清零）。
- links: `constraint` EK-23（SourceClass 不认证 → 不能默认信任）约束 EK-15；`causal` EK-14←EK-15
- evidence: detectors/self_reinforcement.py:150-175；commit 4e95945；tests/test_self_reinforcement_trust.py

### EK-18 CrossTaskContaminationDetector：durable 记忆跨任务读取标记
`tool_observation`/`retrieved_fact`（DURABLE_CLASSES）由任务 A 写入后，任务 B 读取即标记（autogen#7673 scenario 1——"task A 的工具结果静默出现在无关任务 B"）。
- links: `dependency` EK-16（依赖分类注册表）；`subsystem` EK-14（同为语义检测）
- evidence: detectors/cross_task.py docstring；classification.py DURABLE_CLASSES

### EK-34 深度边界设计的完整意图链
#135 限制递归（防 crash）→ 暴露"截断=静默盲区"→ #151 把盲区变成 finding。两次修复形成"边界必须可见"的完整设计闭环。
- links: `causal` EK-06→EK-07→EK-02；`contrast`（初始 fix 引入新洞，第二次 fix 收口）
- evidence: commit c5904bf + 24d1d26 连读

## C. 策略与权威
### EK-08 Policy 声明式 YAML 四要素
`version / default_action / protected_keys / immutable_keys / rules`；rules 为有序列表（on detector + action + min_severity + keys globs）。strict()/tiered()/permissive() 三预设。
- links: `subsystem` EK-09/EK-10；`constraint` EK-08 约束 EK-02（策略决定 Action）
- evidence: policies/policy.py docstring + from_dict

### EK-09 tiered 预设：key 命名空间决定响应等级
credentials.*/permissions.* block injection+secrets；facts.* quarantine；preferences.* redact；tool_results.* block injection/quarantine size；scratch.* quarantine size。文档明示 TTL/revalidation/actor-role gating 是独立 roadmap 项（未实现）。
- links: `contrast` EK-08（全局 vs 分域规则）；`mechanism` EK-10
- evidence: policies/policy.py tiered() docstring

### EK-10 决策引擎：首个匹配规则 + 动作升级
`Policy.decide` 顺序扫描 rules，首个 applies_to（on 匹配 + severity ≥ min + keys globs）生效；无匹配 → default_action。`_decide` 多 verdict 时 `_escalate` 取最高 rank（ALLOW<REDACT<QUARANTINE<BLOCK）。
- links: `causal` EK-10→EK-02/EK-03；`constraint`（规则顺序 = 优先级）
- evidence: policies/policy.py decide；guard.py _decide/_escalate/_ACTION_RANK

### EK-11 ProtectedKeyDetector：glob 只拦写
fnmatchcase 匹配 protected 模式，仅 operation=="write" 触发（delete 走 guard.delete 内联检查）。
- links: `mechanism` EK-04；`constraint` EK-11 约束 EK-04（保护名单决定可删集）
- evidence: detectors/protected_keys.py

### EK-33 GitHub Action 注入防护：输入经 env 传递
action.yml 把 inputs 通过 env 传给 run 脚本而非字符串替换进脚本，防 `${{ }}` 表达式注入（#122，含 test_action_no_expression_in_run.py）。
- links: `subsystem` EK-28（Action 扫描器）；`contrast`（CI 供应链安全实践）
- evidence: commit 6dcc69e；action.yml

## D. 分类与信任梯度
### EK-16 六类 MemoryClass + 晋升图
EPHEMERAL / USER_PREFERENCE_CANDIDATE / VERIFIED_PREFERENCE / RETRIEVED_FACT / TOOL_OBSERVATION / POLICY；`DEFAULT_PROMOTION_GRAPH`：ephemeral→candidate→verified（**requires_verification=True**）、retrieved_fact→tool_observation；其余全部拒绝。来源：armorer-labs 在 microsoft/autogen#7673 提出的 ASI06 缓解模式。
- links: `mechanism` EK-17；`causal` → EK-02（write 分类检查）
- evidence: classification.py；guard.py promote()

### EK-17 untrusted 类永不晋升 trusted
RETRIEVED_FACT / TOOL_OBSERVATION（UNTRUSTED_CLASSES）**永不** promote 到 POLICY / VERIFIED_PREFERENCE——策略变更必须来自受信主体的 POLICY 写。"untrusted sources cannot silently become trusted policy"。
- links: `constraint` EK-17 约束 EK-02（promote 门）；`mechanism` EK-16
- evidence: classification.py docstring + DEFAULT_PROMOTION_GRAPH 注释

### EK-23 SourceClass 是元数据不是认证
`SourceClass`（external_tool/user_input/agent_authored/system/unknown）由调用方声明；events.py 与 self_reinforcement.py 双处明示："does not authenticate a principal / not an authentication or identity assertion"。
- links: `constraint` EK-23 约束 EK-15/EK-16（信任梯度依赖诚实 provenance）
- evidence: events.py SourceClass docstring；self_reinforcement.py docstring

## E. 完整性、快照、事件
### EK-19 canonical_serialize → SHA-256 基线
sort_keys + separators(",",":") + default=str + ensure_ascii=False 稳定 JSON → sha256。formatting 变化不破基线；baseline/verify/clear。
- links: `causal` EK-19→EK-03（read verify）；`subsystem` EK-20（快照同用 hash_value）
- evidence: integrity.py；storage/snapshots.py import

### EK-20 Snapshot.digest 永不验证（诚实边界）
`Snapshot.digest` 仅取证记录，无人重算/比对；rollback 恢复未验证数据。docstring 解释：快照在进程内存中，篡改快照已意味着代码执行，勿把它当完整性检查（#139 明示）。
- links: `contrast` EK-19（基线验证 vs 快照不验证）；`causal` EK-21（快照用于回滚）
- evidence: storage/snapshots.py:15-30；commit 4eca0d5

### EK-21 pre-block 快照 + rollback
`snapshot_on_block=True`（默认）时 BLOCK 前 capture("pre-block")；`rollback(snapshot_id)` 恢复 store 并 emit rollback 事件（metadata 带 digest）。
- links: `causal` EK-02→EK-21；`dependency` EK-21 依赖 EK-20（快照结构）
- evidence: guard.py:250/579-600

### EK-22 SecurityEvent：SIEM-ready 结构化事件
detector/severity/action/key/message/operation/source_class/receipt_uri/source_type/metadata/timestamp/event_id；`to_dict()` 供 SIEM 转发；receipt_uri 指向外部加密签名审计链（如 Ed25519 共签），下游 SOC 可关联 guard 决策与执行收据。
- links: `causal` EK-05→EK-22；`mechanism` EK-30（OTel 转写）
- evidence: events.py SecurityEvent

### EK-30 OTel hook：每决策一 span（Layer 2）
每个 SecurityEvent → 单 OTel span，属性取自 dataclass；SOC 经同 trace context 关联 guard 触发与 agent 决策/收据。lazy import——无 OTel 时降级 stdout。
- links: `subsystem` EK-22；`causal` EK-30→EK-31（观测支撑合规）
- evidence: examples/opentelemetry_hook.py docstring

### EK-31 合规映射的可审计声明
compliance-mapping.md："Every quantitative claim in this document resolves to one of two artifacts. Nothing here is estimated, extrapolated, or rounded up."；同时声明 corpus 自编、非对自适应对手的度量（"What these numbers do not establish"）。
- links: `constraint` EK-31 约束 EK-22（事件须能支撑审计）；`mechanism` EK-30
- evidence: docs/compliance-mapping.md §0

### EK-32 轻量设计：中位延迟 62µs
compliance-mapping：中位加延迟 62µs/记忆操作（单进程、无网络后端）；README 表 59µs（0.2.2 前后版本差）。核心仅 PyYAML 硬依赖。
- links: `contrast` EK-16（ML 检测器可选，默认不载 torch）
- evidence: docs/compliance-mapping.md §0；README benchmark

## F. 集成与基准
### EK-27 MCP server：扫描器包装为工具
FastMCP 包装 AMG 扫描 → MCP 兼容 agent（Claude/GPT/Gemini）实时校验记忆条目；ThreatCategory 六类（prompt_injection/secret_leakage/integrity_tampering/role_hijacking/instruction_override/data_exfiltration）；ScanResult{is_safe, risk_score, threats}。发布走 trusted-publishing workflow（#100）。
- links: `subsystem` EK-28；`mechanism` EK-13（复用泄漏检测）
- evidence: mcp-server/src/amg_mcp_server/server.py

### EK-28 静态防线：scanner + semgrep + SARIF + GitHub Action
scanner/scan.py 按规则集扫描文件（policy 分级 basic/moderate/strict 过滤）；semgrep/agent-memory-unguarded.py 规则；SARIF 输出；action.yml 集成 CI。
- links: `contrast` EK-27（静态 vs 运行时）；`subsystem` EK-33
- evidence: scanner/scan.py；semgrep/；action.yml

### EK-29 集成矩阵：drop-in 中间件与 HITL
LangChain `GuardedChatMessageHistory`（记忆历史包装）+ `MemoryGuardMiddleware`（默认 block on violation）；OpenAI Agents `GuardedAgentContext/GuardedToolOutput/GuardedHandoff`（**block 时进 HITL queue**）；AutoGen/agno/CrewAI/LlamaIndex/mem0 适配；LangGraph 示例（secure_langgraph_memory.py）。
- links: `mechanism` EK-02（都走 write 管线）；`subsystem` EK-35
- evidence: src/agent_memory_guard/integrations/*.py；README integrations 节

### EK-24 AMSB：model-free harness，单一确定性判定
harness 对每 scenario 施加攻击步骤 → 模拟 context reset → 读回 probe key → 唯一规则判定 breach/defended：**canary 是否存活进 recalled value**。"系统不能在写路径显聪明、读路径还毒给 agent"；两人跑同语料同分。
- links: `constraint` EK-25（评分约束）；`mechanism` EK-26
- evidence: src/agent_memory_guard/bench/harness.py docstring
- **Reconciliation 补充（IF-4，独立验证）**：bench/adapters/baseline.py:11-21 明示——`AMGStrictAdapter`（官方推荐 enforcing 配置）**故意不加载** opt-in 的 persistence detector，`AMGHardenedAdapter` 才加 `MemoryPersistenceInjectionDetector` + protected/immutable identity keys；"Reporting both, side by side, is the honest version of 'how good is AMG'"——**基准把默认配置的缺口做成对照实验，不隐藏**。AMSB 实测（本 run 独立复跑）：strict=4/15 breach（全 memory_persistence）、71.4 分 D；hardened=1/15 breach（persist-directive-evasion）、93.9 分 A。

### EK-25 AMSB 评分三分：加权防御率 + FP 惩罚 + 等级封顶
①恶意 scenario 按 severity 权重计防御率；②benign 过拦截（false_positive）扣分（"全拒不是防御"）；③任一 critical 被 breach 等级封顶 C、多个封顶 D（对照 SSL Labs 单 critical CVE 封顶），数值分仍报（封顶可见不静默）。
- links: `causal` EK-24→EK-25；`contrast` EK-24（判定 vs 评分分离）
- evidence: src/agent_memory_guard/bench/scoring.py docstring

### EK-26 基准诚实性：self-submission floor + 披露
自提交行标记（test_adapters.py:66 "amg_adapters_are_marked_self_submission_floor_is_not"）；结果含 benchmark metadata 与 disclosure；markdown 含 grades/disclosure/reproduce line。
- links: `mechanism` EK-25；`constraint` EK-31（对外数字可复现）
- evidence: src/agent_memory_guard/bench/{adapters,report}.py；tests/bench/test_report.py

### EK-35 CLI 一致性：serve --policy 应用到 API guard
#147：`amg serve --policy` 此前不影响 API guard（CLI 与 API 策略分离 bug）；#150：CLI/API 如实报告 blocked input、demo 如实报告结果、LlamaIndex chat store 可用。
- links: `subsystem` EK-29；`causal` EK-35→EK-02（策略一致性）
- evidence: commit b63dfb4 / 3e61797

## 图统计
- EK 数：33（EK-01~EK-35 连续编号，剔除空号）
- 边类型覆盖：mechanism（EK-12↔13、EK-16↔17、EK-24↔26 等）、subsystem（决策管线/检测族/策略族/基准族）、causal（EK-02→03→21、EK-06→07→02、EK-19→03 等）、dependency（EK-18→16、EK-21→20）、constraint（EK-11→04、EK-17→02、EK-23→15/16、EK-31→22）、contrast（EK-01↔08、EK-12↔16、EK-19↔20、EK-24↔25、EK-27↔28）
- 游离 EK：0（全部 ≥1 links）
