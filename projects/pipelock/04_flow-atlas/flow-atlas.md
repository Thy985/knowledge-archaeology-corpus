# 04 · Flow Atlas（七类流）— Pipelock

> 所有 Edge 从真实代码导出，可回溯 symbol/file/condition。28 条 Edge。

## 1. Control Flow（控制流）
| # | Edge | 条件/转换 | 证据 |
|---|---|---|---|
| C1 | Decide 入口按 EventKind 分发 | shell/mcp/file/webfetch/write/tooluse；unknown → Deny | decide/decide.go:122-143 |
| C2 | Airlock tier 分级放行 | None/Soft→放行；Hard→classifyHard；Drain→block；unknown→block | proxy/airlock.go:317-335 |
| C3 | MCP ForwardScannedInput 主循环 | 读→扫→blockedCh 通知→干净转发；defer cancel（kill watch） | mcp/input.go:248-306 |
| C4 | guard 应用→校验→（失败）拒绝启动 | ErrManifestIncomplete/ErrPolicyNarrowed/ErrAlreadyApplied | guard/guard.go:60-90 |
| C5 | adaptive escalation → session 升级 → 后续动作升级 | recordAdaptiveFinding→recordMCPAdaptiveSignal | mcp/input.go:recordAdaptiveFinding |

## 2. State Flow（状态流）
| # | Edge | 状态转换 | 证据 |
|---|---|---|---|
| S1 | AirlockState tier 变迁 | SetTierWithProvenance / TryDeescalate / HalfClose / FullClose | proxy/airlock.go:157-300 |
| S2 | 收据链生命周期 | open→stream→close；torn_tail→recovery_seal；startup 恢复 | receipt/chain*.go, recovery*.go, session*.go |
| S3 | 平行组状态 | group open→inventory→verify→terminal；改签名密钥→关闭组 | receipt/group_*.go；CHANGELOG #1828 |
| S4 | MCP 工具基线状态 | 无基线→捕获 tools/list→绑定→未知工具 warn/block | mcp/input.go:SessionBindingConfig |
| S5 | issuer cookie 状态 | 写 $XDG_STATE_HOME；失败→证据重置→重扫 | CHANGELOG 3.7.0（issuer cookie） |

## 3. Data Flow（数据流）
| # | Edge | 数据路径 | 证据 |
|---|---|---|---|
| D1 | 请求体 → 扫描器管线 → 决策 | URL 全量先于 DNS；prefilter→entropy→DLP | scanner/core.go, prefilter.go；attacks-blocked.md |
| D2 | ActionRecord → canonical → SHA-256 → Ed25519 → Receipt | Sign 路径 | receipt/receipt.go:47-76 |
| D3 | Receipt → flight recorder → AEL 流 → chain/group | emit/emitter.go；256MiB 上限 | CHANGELOG #1872 |
| D4 | MCP 请求 → 扫描 → blockedCh → 错误响应 | 主 goroutine 独占 clientOut 写 | mcp/input.go:blockedCh |
| D5 | 配置 reload → canonical hash → 拒绝/接受 | file watcher 100ms debounce；安全降级拒绝 | docs/configuration.md:15,23 |

## 4. Evidence Flow（证据流）★核心
| # | Edge | 证据路径 | 证据 |
|---|---|---|---|
| E1 | 决策 → typed ActionRecord（含 authority/taint/contract context） | EK-05 | receipt/action.go:113-190 |
| E2 | ActionRecord → canonical JSON（同一 preimage） | EK-06 | receipt/canonical.go |
| E3 | canonical → SHA-256 → Ed25519 签名 → Receipt{Version, ActionRecord, Signature, SignerKey} | EK-01 | receipt/receipt.go:47-76 |
| E4 | Receipt → 发射 → 链/组（open→close；endproof） | EK-11/12/13 | receipt/chain_walk.go, group_verify.go |
| E5 | 验证：VerifyWithKey（版本→记录→签名格式→长度→密钥匹配→canonical 重算→ed25519.Verify） | EK-04 | receipt/receipt.go:84-146 |
| E6 | Ext 袋：捕获 verbatim、不签名、不 consult | EK-01 | receipt.go:Receipt.Ext |
| E7 | 独立 verifier 离线验证（Go/TS/RS/Py） | 跨实现一致性 | CHANGELOG #1828（pipelock-verifier-rs 0.6.5 等） |

## 5. Authority Flow（权威流）
| # | Edge | 权威路径 | 证据 |
|---|---|---|---|
| A1 | 签名密钥来源 → VerifyWithKey 信任锚 | 无 Verify()；外部锚强制 | receipt.go:78-83 |
| A2 | rules 签名三 tier：official keyring → third-party keys → unsigned | TrustPolicy 组合值 | rules/verify.go:30-98 |
| A3 | capability gate：license_feature → EnforceLicenseGate → license.FeatureAgents/VerifyFleetWithOptions | enterprise/config.go, internal/license/fleet_gate.go | capability-manifest.json |
| A4 | 收据组验证权威：group-capable verifier 才可验证组 | PyPI 包仅分片级 | CHANGELOG #1828 |
| A5 | SPIFFE SVID 姿态（posture/svid/svidsidecar） | 姿态绑定 | internal/posture/, svid/ |

## 6. Memory Flow（记忆流）
| # | Edge | 记忆路径 | 证据 |
|---|---|---|---|
| M1 | session taint 跨请求记忆（SessionContaminated/RecentTaintSources/TaskID） | 收据上下文携带 | receipt/action.go；session/taint_classify.go |
| M2 | MCP RequestTracker（转发请求 ID 记忆 → 响应 solicited 校验） | mcp/input.go tracker | mcp/input.go:251-258 |
| M3 | 工具基线记忆（FrozenToolRegistry / ToolBaseline） | 会话内冻结 | proxy/airlock.go:385-447；mcp/baseline.go |
| M4 | 配置/规则状态记忆（reload 拒绝保留旧配置；规则 bundle 当前 live 集） | docs/configuration.md:46 | config/canonical.go |

## 7. Policy Flow（策略流）★治理闭环
| # | Edge | 治理路径 | 证据 |
|---|---|---|---|
| P1 | 决策 → approval（HITL approve） | hitl/approver.go | internal/hitl/ |
| P2 | 决策 → 规则包（rules bundle verify/freshness/transaction）→ enforcement | rules/verify.go, freshness.go, transaction.go | rules/ |
| P3 | 策略 → kill switch → 全局阻断（含 deferred 请求唤醒） | killswitch/api.go, killswitch.go；mcp/input.go watchDeferredKillSwitch | internal/killswitch/；**Reconciliation 补强（独立验证 ERR-3/M-1）**：kill switch 有 ≥7 激活源（config、API、Conductor remote kill、Conductor stale…，killswitch.go:5-7）+ EnvAPIToken 覆盖 + 独立端口豁免（killswitch.go:66,92,106） |
| P4 | 策略 → adaptive escalation → 会话升级 → 后续决策 | EK-17 | decide/escalation.go；proxy/adaptive_upgrade.go |
| P5 | 策略 → config 安全降级拒绝 → 保留旧配置（治理闭环：Decision→Approval→Policy→Enforcement→Future Decision） | docs/configuration.md:15,46 | config/ |

## Flow→KO 交叉校验门
- KO-01 ← E1-E7（证据流全链）；KO-02 ← C2/C4/D5/P5（fail-closed 四实例）；KO-03 ← D1/D4（parser-differential 两传输面）；KO-04 ← S2/S3/E4（链完整性）；KO-05 ← C1-C5 全链；KO-06 ← A2/P2（组合值）；KO-07 ← C1/D1（强度序位）；KO-08 ← D1/E1（pre-side-effect）。无矛盾。
