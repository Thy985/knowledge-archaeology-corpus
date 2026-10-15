# 01 · Project Layer — Pipelock 项目地图

## 1.1 定位与使命
- 开源 AI agent 防火墙：网络+工具代理，中介 HTTP/WebSocket/MCP/A2A 流量（+TLS 拦截时 CONNECT 隧道内容），检测 secret exfiltration / prompt injection / SSRF / tool poisoning / risky tool-call chains；产出 mediator-signed action receipts（外部验证）（README.md / AGENTS.md）
- 目标用户/场景：coding agent（Claude Code/Codex/Cline/OpenCode/Pi/Zed/Cursor/VS Code/JetBrains）+ 框架（OpenAI Agents SDK/ADK/AutoGen/CrewAI/LangGraph）；部署形态：proxy / MCP wrapper / sandbox / contain（host containment）/ cluster（Helm chart）
- 商业模型：Core Apache-2.0 永久免费；enterprise/ ELv2（agents profiles/fleet/conductor/dashboard/coverage certs）；CHARTER.md "License Promise"（一切已免费者保持免费）

## 1.2 架构（四层分离）
```
[Agent 工作负载] → [代理/中介层 proxy·airlock + mcp] → [扫描检测 scanner/shield/rules] → [决策 decide]
                                                                                        │
                                        [执行强制 guard/sandbox/contain · Linux Landlock] ←┘
                                                                                        │
                                        [证据链 receipt/emit/anchor/signing → flight recorder] → [独立 verifier 离线验证]
```
检测（detect）≠ 决策（decide）≠ 执行（enforce）≠ 证据（evidence）——四层职责分离，互不信任（架构从 AGENTS.md capability manifest + 包结构导出）。

## 1.3 核心模块（证据：internal/ 包 + 文件）
| 子系统 | 代表文件 | 职责 |
|---|---|---|
| 代理中介 | proxy/airlock.go, proxy/agent.go | Airlock 会话状态机（四 tier）；动作分类；FrozenToolRegistry |
| MCP/A2A | mcp/input.go, mcp/a2a_*.go, mcp/mcpwrap | JSON-RPC 输入扫描、响应 ID 校验、session 工具基线绑定、A2A 卡签名/深度/溢出防护 |
| 扫描检测 | scanner/core.go, scanner/*.go（entropy/known_secret/databudget/ratelimit/redaction…） | DLP/注入/SSRF/工具投毒多扫描器管线 |
| 内容防护 | shield/shield.go（HTML/SVG/XML 重写、nonce、tracking 剥离） | 隐藏指令/工具投毒的内容层防护 |
| 决策 | decide/decide.go（Decide → 六类 payload 子决策）、decide/escalation.go、decide/cross_agent.go | 内容感知边界决策 + 自适应升级 |
| 执行强制 | guard/guard.go（Landlock 清单、ErrManifestIncomplete 等 fail-closed）、sandbox/ | OS 级文件/进程/网络强制 |
| 证据链 | receipt/（action.go, receipt.go, chain*, group*, emitter*, session*, sanitize, canonical）、evidence/、emit/、signing/、anchor/（rekor） | 签名收据 + 链/平行组 + 发射 + 锚定（Rekor） |
| 规则治理 | rules/（verify, loader, keyring, freshness, transaction, version） | 签名规则包（官方/第三方/unsigned 三 tier） |
| 配置治理 | config/（canonical, action, action_divergence）、configs/*.yaml | reload 拒绝安全降级；严格模式 |
| 治理原语 | killswitch/, hitl/approver.go, authlimit/, posture/, svid/ | kill switch、人工介入、鉴权限流、SPIFFE 姿态 |
| 覆盖证明 | coveragecert/, coveragecertverify/, fleetreceipt/ | per-agent 覆盖证书 + 车队收据 |
| 能力清单 | internal/capabilitymanifest/ → docs/security/capability-manifest.json | 27 项能力 × gate（free/license_feature） |

## 1.4 生命周期与状态
- **Airlock 会话**：None→Soft→Hard→Drain 四 tier；TryDeescalate 定时降级；HalfClose/FullClose；in-flight 计数（airlock.go）
- **收据链生命周期**：chain open → stream（head/typed chain）→ group close（group_close/terminal）→ recovery（torn_tail/recovery_seal/startup）；session_control（issuer cookie）；AEL 流 256MiB 上限；10-min stop budget（CHANGELOG 3.7.0 #1909）
- **配置生命周期**：file watcher/SIGHUP 100ms debounce reload；安全降级拒绝保留旧配置（docs/configuration.md）

## 1.5 依赖与入口
- Go 1.26+；直接依赖：common-fate/httpsig, go-jose/v4, landlock-lsm/go-landlock, spiffe/go-spiffe/v2, santhosh-tekuri/jsonschema, gobwas/ws, goccy/go-yaml, spf13/cobra, rs/zerolog, prometheus, sentry, OTLP proto（go.mod）
- 入口：cmd/pipelock（main.go + enterprise.go）；14 个命令含 pipelock-verifier / pipelock-verifier-wasm / pipelock-replay-capture；独立 verifier 生态（@pipelock/verifier-ts、pipelock-verifier-rs、pipelock-verify PyPI）

## 1.6 测试体系
- 2496 Go 测试文件；CI ci.yaml（Go 1.26/1.27）+ security.yaml + hardening.yaml + continuous-gauntlet.yaml（pin agent-egress-bench corpus v2.14.0，不自动发布分数）+ clusterfuzzlite + tests/pentest.sh
- 对抗性测试族：forgery_poc_test.go / strict_verify_test.go / failclosed_linux_test.go / adaptive_*_test.go / bypass-resistance.md

## 1.7 权限与治理机制
- capability manifest（go generate 生成，27 项；gate_coverage/gate_exclusions）；license gate：enterprise/config.go EnforceLicenseGate → license.FeatureAgents / VerifyFleetWithOptions（internal/license/fleet_gate.go）
- rules 包更新安全：TrustPolicy（TrustedKeys + TrustEmbeddedKeys 组合值——单独使用任一都是 bug）、CheckSignerPinning、bundle verify 三 tier（verify.go）
- CHARTER.md founder-led；GOVERNANCE.md/SECURITY.md/CONTRIBUTING.md；docs/security/*（conductor/audit-packet/per-deployment-ca threat models、tier-gating-audit-matrix、key-rotation-runbook、monotonic-state-threat-boundary）
