# 02 · Engineering Knowledge（EK Graph，宽底座）— Pipelock

> 每条 EK 标注证据（file:symbol）与 links（六类边：mechanism/subsystem/causal/dependency/constraint/contrast）。35 条。独立价值，不因"不够抽象"删除。

## A. 决策与证据核心（receipt/decide/emit/verify）

**EK-01 签名只覆盖 canonical action record，Ext 永不 consult（Fact）**
证据：internal/receipt/receipt.go:47-76（Sign 覆盖 canonicalActionRecord 的 SHA-256）；Receipt.Ext 注释（"UNSIGNED, advisory… NEVER consulted by verification"）。
links: mechanism(EK-02, EK-06); constraint(EK-03); subsystem(EK-05)

**EK-02 Verify() 故意不可用——外部信任锚强制（Fact）**
证据：receipt.go:78-83（`Verify` 直接 return error 引导 VerifyWithKey）。
links: mechanism(EK-01); causal(EK-01→EK-02→EK-04); contrast(EK-07)

**EK-03 Unmarshal fail-closed 三 parser-differential 面（Fact）**
证据：receipt.go:195-232（RejectDuplicateKeys + DisallowUnknownFields + trailing tokens；legacy detected_patterns 走 unmarshalLegacyDetectedPatterns）。
links: constraint(EK-01, EK-06); mechanism(EK-27); subsystem(A2)

**EK-04 VerifyWithKey 全链校验（版本/记录/签名格式/长度/密钥匹配/canonical 重算）（Fact）**
证据：receipt.go:84-146。
links: causal(EK-02→EK-04); dependency(EK-06)

**EK-05 ActionRecord 是 typed action model（8+1 类型 × side-effect × reversibility × authority/taint/contract context）（Fact）**
证据：receipt/action.go:113-190（ActionRecord 字段族：ActionType 9 类、SideEffectClass 5 类、Reversibility 4 类、DelegationChain、SessionTaintLevel、ContractWinningSource…）。
links: subsystem(EK-01, EK-06); causal(EK-05→EK-09→EK-10); constraint(EK-08)

**EK-06 canonical 规范化是签名与验证的同一 preimage（Fact）**
证据：receipt/canonical.go + canonical_parity_test.go + canonical_golden_test.go（canonical JSON，marshal 侧 NormalizeReplacementEscapes 与 verify 侧 re-encode 一致）。
links: mechanism(EK-01); dependency(EK-03, EK-04); constraint(EK-07)

**EK-07 Sign（Marshal）与 Verify（Unmarshal）严格性解耦——消毒只发生在 sign 前（Fact）**
证据：receipt/sanitize.go（sanitize_before_sign_test.go）；receipt.go:158-194（Marshal NormalizeReplacementEscapes；注释"closes the gaps on the verify side without touching the signing input"）。
links: contrast(EK-02); mechanism(EK-06); subsystem(A)

**EK-08 "签名可验证 ≠ 决策可验证"——签名只证明判定内容被签，不证明判定正确（Observation→Hypothesis）**
证据：receipt 包注释（"Receipt signatures prove only that the action record was signed by a key"）；README（"reviewer can verify what Pipelock decided outside the agent runtime"）；docs/receipts/versioning.md。
links: constraint(EK-01, EK-02); causal(EK-08→EK-24); contrast(EK-25)

**EK-09 ActionID 生成态与调用者提供态分离——生成 ID 带 origin proof 排除扫描，调用者 ID 是内容被扫描（Fact）**
证据：receipt/action.go:NewActionID（receiptcontent.NewGeneratedID 带 origin proof，被 content boundary 排除；caller 提供的 ID 视为内容）；CHANGELOG 3.7.0 #1872/1886/1897（收据内容扫描有界重组）。
links: mechanism(EK-05); causal(EK-09→EK-10); contrast(EK-27)

**EK-10 收据内容边界（receipt-content-boundary）——完整 secret 仍签前检查；重组有界（五片/四片乱序/过宽三片排除）（Fact）**
证据：CHANGELOG 3.7.0（receipt content scanning explicit bounded search #1872/1886/1897）；docs/guides/receipt-verification.md#receipt-content-boundary。
links: causal(EK-09→EK-10); constraint(EK-06); subsystem(A)

**EK-11 收据链 open prefix 不再算 complete（3.7.0 破坏性变更）（Fact）**
证据：CHANGELOG 3.7.0 #1892（VALID PREFIX, END UNPROVEN exit 3；clean recorder run 无 end marker 也 exit 3；`--whole-recorder`）；receipt/chain_walk.go + recovery_*。
links: causal(EK-10→EK-11); constraint(EK-12); subsystem(A)

**EK-12 平行收据组（receipt_chains 1~32）——有效分片不是完整组证据；改签名密钥关闭组（Fact）**
证据：CHANGELOG 3.7.0 #1828/1838/1863（group-capable verifier 才组验证；PyPI pipelock-verify 0.4.1 只验分片不验组）；receipt/group_*.go（group_verify.go, group_inventory_*.go）。
links: causal(EK-11→EK-12); mechanism(EK-13); dependency(EK-02)

**EK-13 组关闭重哈希全部证据 + 10-min stop budget（Fact）**
证据：CHANGELOG 3.7.0 #1909（300k req / 8 chains ≈ 15s@8CPU、109s@2CPU）；contain install/Helm chart 设 10 min。
links: mechanism(EK-12); causal(EK-13→EK-14)

**EK-14 AEL 流 256MiB 验证上限——写入器无上限，超限 run 无法发布可验证 close（Fact）**
证据：CHANGELOG 3.7.0（native AEL 256 MiB verifier ceiling；200k req/8 chains ≈ 28MB 最大流 → 上限≈1.9M 请求量级）。
links: causal(EK-13→EK-14); constraint(EK-11); subsystem(A)

## B. 代理中介（airlock/proxy/mcp）

**EK-15 Airlock 四 tier（None/Soft/Hard/Drain）+ 未知 tier fail-closed block（Fact）**
证据：proxy/airlock.go:64-71（AirlockTierOrder）、317-335（ClassifyAction：Drain 全阻塞；default 未知 tier → "airlock: unknown tier"）。
links: mechanism(EK-16); causal(EK-15→EK-17); subsystem(B)

**EK-16 Hard tier 只允许 read-only 方法 + fetch（Fact）**
证据：airlock.go:72（readOnlyMethods）+ classifyHard（TransportFetch 只读放行）。
links: causal(EK-15→EK-16); subsystem(B)

**EK-17 Adaptive escalation：威胁信号记录 → 会话升级 → 有效 action 升级（Fact）**
证据：mcp/input.go:recordAdaptiveFinding（adaptiveCfg.Enabled → recordMCPAdaptiveSignal → EscalationParams）；proxy/adaptive_upgrade.go；decide/escalation.go。
links: causal(EK-15→EK-17→EK-18); mechanism(EK-22); subsystem(B, D)

**EK-18 Session taint 跨请求传播 + taint-aware 政策升级（Fact）**
证据：receipt/action.go（SessionTaintLevel/SessionContaminated/RecentTaintSources/TaintDecision）；internal/session/taint_classify.go + risk.go。
links: causal(EK-17→EK-18); constraint(EK-19); subsystem(A, B)

**EK-19 MCP 输入扫描：DLP+注入双扫 → 干净才转发；被 block 请求 ID 经 channel 回主 goroutine 防写竞争（Fact）**
证据：mcp/input.go:248-306（ForwardScannedInput；blockedCh 设计："so the main goroutine (which owns clientOut writes) can send error responses without concurrent write races"）。
links: mechanism(EK-20); subsystem(B)

**EK-20 MCP 响应 ID 校验（RequestTracker：响应 ID 必须是被 solicited 的请求 ID）（Fact）**
证据：mcp/input.go:251-258（tracker 记录每个转发请求 ID，response-side 校验 solicited）；mcp/input.go 相关跟踪实现。
links: causal(EK-19→EK-20); constraint(EK-21); subsystem(B)

**EK-21 MCP malformed JSON / 无效 UTF-8 / duplicate keys 拒绝——即便 on_parse_error: forward；被拒响应不更新追踪状态（Fact）**
证据：CHANGELOG 3.7.0 #1888/1889（schema-bound JSON requires canonical keys；MCP 上 malformed JSON、invalid UTF-8、duplicate keys 拒绝 despite on_parse_error: forward；rejected responses don't update tracking state）；**Reconciliation 补强（独立验证 ERR-1）**：OnParseError 默认即 ActionBlock（internal/config/normalize.go:371-374 "Always default OnParseError (fail-closed) regardless of enabled state"）——fail-closed 是默认配置而非仅文档声称。
links: mechanism(EK-27); constraint(EK-03, EK-20); subsystem(B)

**EK-22 A2A 卡签名/深度/溢出三防护 + session 工具基线绑定（FrozenToolRegistry）（Fact）**
证据：proxy/a2a_card_signature_e2e_test.go / a2a_depth_test.go / a2a_overflow_test.go；proxy/airlock.go:385-447（FrozenToolRegistry Freeze/IsFrozen/IsToolAllowed）；mcp/baseline.go + SessionBindingConfig（UnknownToolAction warn/block）。
links: mechanism(EK-17); causal(EK-22→EK-23); subsystem(B)

**EK-23 工具基线（tool inventory）绑定：tools/list 首次响应捕获基线，之后 tools/call 对照校验（Fact）**
证据：mcp/input.go:SessionBindingConfig（Baseline from first tools/list；UnknownToolAction/NoBaselineAction warn|block）。
links: causal(EK-22→EK-23); constraint(EK-20); subsystem(B)

## C. 防护执行（guard/sandbox/shield）

**EK-24 guard 以"拒绝启动"表达 fail-closed（manifest 不完整/策略被收窄/二次应用 → 拒绝，不是收窄授予）（Fact）**
证据：guard/guard.go:60-90（ErrManifestIncomplete/ErrPolicyNarrowed/ErrAlreadyApplied；"a declared path is a statement that the workload needs it… this is a refusal, not a caveat"）。
links: causal(EK-08→EK-24); mechanism(EK-25); subsystem(C)

**EK-25 Landlock 限制只能叠加不能解除 → 二次应用 = 策略自交窄化永久（Fact）**
证据：guard/guard.go ErrAlreadyApplied（"Landlock restrictions stack and cannot be lifted"）；apply_linux.go。
links: mechanism(EK-24); constraint(EK-24); subsystem(C)

**EK-26 sandbox 分级降级：user namespace 缺失 → 仅 HTTP(S)_PROXY 尽力隔离；seccomp 仅 linux/amd64；Landlock ABI 9/6（Fact）**
证据：AGENTS.md capability manifest（sandbox 段："Without them, best-effort network isolation only sets HTTP(S)_PROXY; seccomp adds restrictions only on linux/amd64; Unix socket isolation uses Landlock ABI 9 for pathname sockets and ABI 6 for abstract sockets"）；CHANGELOG 3.7.0 #1813/1890（strict ARM64 below ABI 9 refuse）。
links: contrast(EK-24); constraint(EK-28); subsystem(C)

**EK-27 shield 内容重写（HTML/SVG/XML）——nonce 注入 + tracking 剥离 + script mask（Fact）**
证据：shield/shield.go:207-392（Rewrite/RewriteWithNonce/rewriteSVG/maskHTMLScripts/stripTracking）。
links: mechanism(EK-03, EK-21); causal(EK-27→EK-28); subsystem(C)

**EK-28 隐藏指令/工具投毒的探测 + 响应完整性：Partial/Range/cache-validator 请求剥离、origin 206/304 → response_incomplete（Fact）**
证据：CHANGELOG 3.7.0 #1809（"Partial and cache-validating HTTP responses must remain inspectable… refuse origin 206 or 304 replies as response_incomplete. Browser Shield exemptions do not bypass that refusal."）；shield/patterns.go + hidden_trap_structure_test.go；**Reconciliation 补强（独立验证 ERR-2）**：response_incomplete 是三处全局路径——forward.go:2368、intercept.go:2153、response_conditions.go:40（browser_cache 触发 `replaceWithBlockReason(resp, ResponseIncomplete, "browser_cache")`），非仅 forward/intercept。
links: causal(EK-27→EK-28); constraint(EK-24); subsystem(C)

## D. 扫描检测（scanner/rules）

**EK-29 DLP 管线在 DNS/连接之前扫完整 URL——secret 永不离开代理（Fact）**
证据：docs/attacks-blocked.md（"DLP patterns run on the full URL before any DNS resolution or network connection. The key never leaves the proxy."）；scanner/core.go + prefilter.go。
links: causal(EK-29→EK-30); subsystem(D)

**EK-30 Evidence.Action 语义：block/warn + core flag（内置核心凭据底线，enforce 关闭也 block）（Fact）**
证据：decide/decide.go:100-118（Evidence.core 注释："marks a finding from the built-in core credential floor, which blocks even when enforce is off"）；docs/configuration.md:69（enforce=false 时 SSRF/fail-closed 传输检查/自适应升级仍可 block）。
links: causal(EK-29→EK-30→EK-31); constraint(EK-33); subsystem(D, E)

**EK-31 buildDecision：无证据→allow；有证据→strictest action；core 优先（Fact）**
证据：decide/decide.go:buildDecision（"Find the strictest action across all evidence" → policy.StricterAction）；config/action.go StrongestAction（block=6 > defer=5 > ask=4 > redirect=3 > strip=2 > warn=1；allow/forward=0 永不被选中）。
links: causal(EK-30→EK-31); subsystem(D, E)

**EK-32 rules 签名包三 tier（official/third-party/unsigned）+ TrustPolicy 组合值（Fact）**
证据：rules/verify.go:30-98（VerifyResult.Tier；TrustPolicy{TrustedKeys, TrustEmbeddedKeys}——"using either without the other is always a bug"；CheckSignerPinning）。
links: mechanism(EK-34); constraint(EK-33); subsystem(E)

**EK-33 规则包解析失败拒绝 reload（任何模式）——瞬态 bundle 失败不能静默削弱覆盖（Fact）**
证据：docs/configuration.md:46（"A reload is also rejected, in any mode, when a rule-bundle resolution error… would drop detection rules that are currently live"）；strict mode 下 clean deletion 拒绝除非 rules.allow_degraded。
links: causal(EK-32→EK-33); constraint(EK-30, EK-24); subsystem(D, E)

**EK-34 配置 reload：安全降级拒绝保留旧配置；fields 需 restart 标记；100ms debounce（Fact）**
证据：docs/configuration.md:15,23（file watcher/SIGHUP 100ms debounce；"A rejected reload keeps the running configuration active"；strict mode + flight_recorder.require_receipts 拒绝安全降级）。
links: mechanism(EK-32, EK-33); subsystem(E)

## E. 治理与配置（config/capability/license/killswitch）

**EK-35 capability manifest 27 项 + gate 映射（free/license_feature + enforcement source symbol）（Fact）**
证据：docs/security/capability-manifest.json（27 capabilities；gate.kind free | license_feature；enforcement.source.file/symbol 可追溯：enterprise/config.go EnforceLicenseGate → license.FeatureAgents；proof file internal/license/fleet_gate.go VerifyAgentsWithOptions）。
links: constraint(EK-36); subsystem(E)

**EK-36 Core credential protection 在 audit 模式也 block；fully-redacted core credential 只转发到配置的 trusted host（Fact）**
证据：CHANGELOG 3.7.0 #1798/1817（"Core credential protection blocks in audit mode. Ordinary policy still warns. On HTTP request bodies, a fully redacted core credential forwards only to a configured request-body trusted host, in audit and enforcement modes."）。
links: mechanism(EK-30); constraint(EK-35); subsystem(E)

**EK-37 JWT cookie 走 header DLP policy；issuer-bound cookie 经 TLS 拦截保持 scoped allowance（Fact）**
证据：CHANGELOG 3.7.0 #1837（"A JWT in a Cookie header now follows the configured action and core floor instead of always warning; an issuer-bound cookie observed through TLS interception keeps its scoped allowance."）；session/control.go（issuer cookie state home $XDG_STATE_HOME）。
links: causal(EK-37→EK-38); subsystem(E)

**EK-38 issuer-cookie 证据需可写 state home——read-only rootfs 失败则证据重置重扫（Fact）**
证据：CHANGELOG 3.7.0（"the write fails, the proxy logs issuer cookie state write failed and the evidence resets on restart; those cookies are then scanned again"）。
links: causal(EK-37→EK-38); subsystem(E)

**EK-39 规则更新锁/交易（lock/transaction/merge）+ version/semver 校验（Fact）**
证据：rules/lock.go、rules/transaction.go、rules/merge.go、rules/version.go（+对应 _test）。
links: mechanism(EK-32); subsystem(E)

**EK-40 hostile-model 预设：零 guardrail 模型 → 全防御层 + 歧义决策全 block（Fact）**
证据：configs/hostile-model.yaml（"Assumes the model has ZERO safety guardrails… This preset enables every defense layer and sets all ambiguous decisions to block. The model won't protect you. The network layer will."）；entropy_threshold 3.0。
links: contrast(EK-31); causal(EK-40→EK-31); subsystem(E)

---
## EK Graph 图论指标
- 节点 35；有向边（causal/dependency/constraint）与无向边（mechanism/subsystem/contrast）合计 63（统计含 subsystem 内聚边）；平均出边 ≥1 ✓
- 游离 EK：0（每条 ≥2 links）✓
- 关键因果链（R2 候选）：EK-01→EK-02→EK-04（签名→外部锚→验证）；EK-05→EK-09→EK-10→EK-11→EK-12→EK-13→EK-14（动作→ID 边界→内容边界→链完整性→平行组→关闭→上限）；EK-29→EK-30→EK-31（扫描→证据→决策）；EK-15→EK-17→EK-18（tier→升级→taint）
