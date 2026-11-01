# 05 · Candidates（未验证假说 / 跨项目模式 / 不确定结论）

> 与 KO 严格区分：以下内容当前证据不足以成立知识，保持 Hypothesis 状态。

## 跨项目假说
**C1. 证据外部化（mediator-signed）可作为 agent 安全治理的通用契约层**
- 假说：把"决策证据由边界外签名"推广为 agent 治理框架的通用要求（RAMPART 观测 + Pipelock 收据 + guardian 强制），可能形成统一的"agent 行为见证"标准面。
- 当前证据：Pipelock S5 实现；RAMPART/guardian 为同方向不同实现（S8 部分）。
- 缺失证据：跨项目同构验证（同一模式在三项目各自独立成立）；业界采用度。
- 验证路径：将 receipt chain + 独立 verifier 模式应用于第二个强制型项目；或观察 SCITT/in-toto（docs/specs 已有 scitt-agent-action-statement-v0.1 / in-toto-agent-action-receipt-v0.1 映射文档——Pipelock 自身已朝此方向走）。

**C2. 决策签名 ≠ 内容签名（转发内容未被中介签名）**
- 观察：MCP"干净转发"的请求内容本身没有逐字节签名（收据签名只覆盖 ActionRecord）；若要求内容级不可抵赖，转发 payload 需要内容哈希进收据。
- 当前证据：receipt 包只签名 canonical action record（EK-01）；docs/specs/evidence-provenance-transform-v4/v5 暗示内容转换有 provenance 但非逐字节。**Reconciliation 补强（独立验证 M-4）**：plain CONNECT 无 TLS 拦截时按 hostname/URL 级扫描（README.md:35），forward.go:215 注释 "hostname-only projection" 是代码级证据——中介签名的真实覆盖范围 = "决策"而非"内容"。
- 影响：对"完全可验证中介"声称的边界（README 说"verify what Pipelock decided"，未声称 verify 转发内容）。

**C3. 工具投毒防护依赖会话基线稳定性**
- 假说：FrozenToolRegistry/工具基线绑定在工具集动态变化（插件安装/更新）时可能产生误阻断或绕过窗口。
- 当前证据：SessionBindingConfig 的 NoBaselineAction 与 UnknownToolAction 显式区分（warn/block 可配）；未发现对基线漂移的处理文档。
- 验证路径：搜索 tools/list 变更后的行为测试（baseline 更新路径）。

## 暂定模式（单项目证据，跨项目 pending）
**C4. "拒绝而非收窄"在文件系统强制中的代价**
- guard 的 ErrPolicyNarrowed 拒绝启动，但 ops 侧代价是：manifest 声明路径与 Landlock 实际可达性不一致时直接不启动（高可用部署需预校验）。与"fail-closed 以拒绝表达"（KO-02）同源，但可运维性权衡未在仓库内闭环（无 degraded-contain 模式）。

**C5. 组级证据的验证能力分层是产品约束而非漏洞**
- PyPI 包仅分片验证 vs TS/RS/Go 组验证：工具生态的证据能力差异会诱导"验证了分片就以为验证了组"的误用；Pipelock 文档显式警告，但未强制（无法从产品层阻止用户误用弱验证器）。

## 不确定结论（NEEDS_HUMAN_REVIEW 候选）
**C6. 收据机制与 fleet/conductor 的权威汇聚**
- fleetreceipt/fleet_receipt-v1.md + conductor 的存在暗示多节点收据汇聚；但汇聚后的组权威（谁有权宣布车队级事实）未在核心包内闭环（enterprise/ 代码未考古）。
- 影响：跨节点证据完整性的权威流（Authority Flow A5 之外）需 enterprise 代码确认。

**C7. AEL 256MiB 上限与 receipt_chains>1 的运维耦合**
- 官方把"restart on a schedule"作为建议而非强制——高频长跑部署存在"run 不可验证"的真实风险窗口（官方文档承认）。

**C8. hostile-model 预设的 block 语义强度**
- "sets all ambiguous decisions to block"——"ambiguous"的判定边界未在 preset 内显式化（依赖各扫描器 severit 映射）；极端模型下可能把警告级也升级为 block（预期行为）或漏掉未标注 ambiguous 的路径（未验证）。

---
## 反例预算（每个 L3+ KO ≥3 定向反例，见 06）
- KO-01 反例：C2（决策签名≠内容签名）、无签名路径（plain CONNECT hostname-only 扫描）、sandbox 分级降级（无中介签名仍运行）。
- KO-02 反例：EK-26 分级降级（有文档的显式降级）、non-strict 模式允许 bundle 降级（rule_bundle_degraded 事件）、enforce=false 的 warn 模式。
- KO-03 反例：Ext 袋是"唯一容忍的未知字段"——结构性例外；legacy detected_patterns 兼容路径（unmarshalLegacyDetectedPatterns）是历史例外。
- KO-04 反例：PyPI 弱验证器、replay profiles v4/v5 pin 字节解码的跨验证器一致性依赖。
