# Snapshot Artifact · ARCH-2026-10-16-001 · Pipelock

## 快照事实（来自仓库实际内容，git 实测）

| 项 | 值 | 来源 |
|---|---|---|
| repository | https://github.com/luckypipewrench/pipelock.git | job_manifest |
| HEAD commit | `8ade100ce6fc48f565e916c12b462e006d699a27` | git log -1 |
| commit msg | "fix: correct v3.7.0 docs and verify playground kits before publishing (#1912)" | git log -1 |
| author / date | Josh · 2026-10-15 10:04:58 -0400 | git log -1 |
| branch | main（origin/main） | git branch |
| version | v3.7.0（2026-10-14 CHANGELOG） | CHANGELOG.md |
| license | Core Apache-2.0 + enterprise/ ELv2 | README/LICENSE/AGENTS.md |
| module | github.com/luckyPipewrench/pipelock | go.mod |
| go | 1.26.0（CI 测 1.26/1.27） | go.mod / AGENTS.md |
| 规模 | 全 repo 5804 文件；Go 3932 文件 / 1,423,579 行（含生成与测试）；核心：cmd 41 文件/12,474 行 + internal 1189 文件/380,608 行（排除 _test/testdata） | find/wc 实测 |
| 测试 | Go 测试文件 2496；CI: ci.yaml / continuous-gauntlet.yaml / security.yaml / hardening.yaml / clusterfuzzlite.yaml；tests/pentest.sh | find / .github/workflows |

## 项目基础地图

### 定位（README/AGENTS.md/CHARTER.md）
Open-source AI agent firewall for **Verifiable Egress Control**：网络与工具代理，中介 AI-agent HTTP/WebSocket/MCP/A2A 流量（+TLS 拦截时 CONNECT 隧道内容），扫描 **secret exfiltration / prompt injection / SSRF / tool poisoning / risky tool-call chains**；产出 **mediator-signed action receipts**（内容感知边界决策的可验证证据，reviewer 在 agent 信任边界之外验证）。自我描述："agent 不能成为自身安全权威"——决策证据由中介（mediator）外部签名。Works with：Claude Code / Codex / Cline / OpenCode / Pi / Zed / Cursor / VS Code / JetBrains / OpenAI Agents SDK / ADK / AutoGen / CrewAI / LangGraph。

### 入口（cmd/，14 命令）
- `pipelock`（主命令：run / mcp proxy / contain / sandbox / exec / demo / init / verifier 等，cmd/pipelock/main.go + enterprise.go）
- `pipelock-verifier` / `pipelock-verifier-wasm`（独立验证器；另有 @pipelock/verifier-ts、pipelock-verifier-rs、pipelock-verify PyPI）
- `license-service`、`pipelock-replay-capture`、playground 系列（broker/demo/live/loadtest/llm-agent/toyagent/webtool）、release-images/release-manifest

### 核心模块（internal/，按体积与职责）
| 包 | 职责（文件证据） |
|---|---|
| `proxy/`（6.8M） | 代理核心：airlock.go（中介主循环）、agent.go、a2a_*.go（A2A 卡签名/深度/溢出防护）、adaptive_upgrade.go、airlock_tunnel_recorder.go |
| `mcp/`（6.3M） | MCP 协议层：mcpwrap、input scanning、typed schema canonical keys（#1888） |
| `scanner/`（2.9M） | 内容扫描：secret/prompt injection/SSRF/工具投毒检测器 |
| `config/`（2.8M） | 配置解析 + schema（configs/*.yaml：audit/balanced/strict/hostile-model/claude-code/cursor/generic-agent） |
| `receipt/`（1.9M） | **签名收据核心**：action.go/receipt.go/chain*.go（链）/group*.go（平行组）/emitter*.go（发射器）/session*.go/verify/sanitize.go/canonical.go |
| `contract/`（1.8M） | 合约/收据 schema 校验 |
| `recorder/`（1.1M） | flight_recorder：证据记录（AEL 流、256MiB 验证上限 #1872） |
| `sandbox/`（840K） | 单进程沙箱（Linux user namespace + Landlock ABI 9/6） |
| `guard/`（460K） | **执行防护**：guard.go/apply_linux.go/exec/execution_proof.go/failclosed_linux_test.go（fail-closed 路径） |
| `shield/`（280K） | **内容防护**：shield.go/patterns.go/css/svg/xml 解析（隐藏指令/工具投毒） |
| `rules/`（464K） | 规则签名链：verify.go/loader/keyring/freshness/transaction（规则更新安全） |
| `decide/` | 决策层：decide.go/cross_agent.go/escalation.go/extract.go/signal.go（内容感知边界决策） |
| `audit/`（468K） | 审计日志（含 core credential protection blocks #1798） |
| `signing/`/`anchor/`/`emit/`/`evidence/`/`egressevidence/`/`coveragecert/`/`coveragecertverify/`/`verification/` | 签名/锚定/证据发射/覆盖证书链 |
| `killswitch/`/`hitl/`/`authlimit/`/`posture/`/`posturebinding/`/`svid/`/`svidsidecar/`/`commitmentkey/`/`identitykey/` | 治理原语：kill switch、人工介入（HITL）、鉴权限流、姿态绑定、SPIFFE SVID |
| `capture/`/`replaycapture/`/`recorder/` | 抓包/回放（replay profiles v4/v5 pin 字节解码 #1888） |
| `aarp/`/`addressprotect/`/`filesentry/`/`seedprotect/`/`gitprotect/`/`processexec/` | 特定攻击面防护（地址/文件/种子/进程执行） |
| `capabilitymanifest/` | 能力清单（docs/security/capability-manifest.json 生成） |

### 核心数据结构（初步，考古深化）
- **Action Receipt**（receipt/action.go）：mediator 签名的决策证据——签名链（chain/group/typed_chain/session_stream）、证据规范（bare_evidence/base_snapshot）、规范化（canonical.go）、签发前消毒（sanitize.go：sign 前 sanitize，secrets 不落入签名字段）
- **Policy/Rules**（rules/）：签名规则集（keyring/verify/freshness/transaction/version）
- **Decide Signal**（decide/signal.go）：跨 agent 决策信号（cross_agent.go）+ 升级（escalation.go）
- **Airlock**（proxy/airlock.go）：代理会话/隧道记录（airlock_tunnel_recorder.go）
- **Coverage Cert**（coveragecert/）：覆盖证书 + 验证（coveragecertverify/）

### 状态与生命周期
- v3.7.0（10-14 发布）；GA 成熟（CNCF Landscape Security & Compliance；OpenSSF Scorecard + Best Practices badge）
- 决策/收据生命周期：receipt 链 open→stream→close（group_close/torn_tail/recovery_seal）；session_control（issuer cookie）；AEL 流 256MiB 上限；关闭 10-min stop budget（#1909）
- 版本治理：CHANGELOG Keep a Changelog + SemVer；`pipelock contain upgrade`/`rules upgrade` 受控

### 测试体系
- 2496 Go 测试文件；`continuous-gauntlet.yaml`（对 pin 的 agent-egress-bench corpus commit 持续考试，不自动发布分数）
- bench（agent-egress-bench 独立仓库 corpus）+ benchmark/ 目录
- security.yaml / hardening.yaml / clusterfuzzlite（OSS-Fuzz 系）/ tests/pentest.sh
- 大量对抗性测试：bypass-resistance.md / adaptive_*_test.go / forgery_poc_test.go / strict_verify_test.go / failclosed_linux_test.go

### 配置
- configs/*.yaml（audit/balanced/strict/hostile-model/claude-code/cursor/generic-agent）+ docs/configuration.md + docs/policy-spec-v0.1.md + schemas/
- 模式：Enforcement / Learning / Permissive（对照 MXC 三模式；docs/contain-cli.md）
- 环境：claude-code.yaml/cursor.yaml 针对特定 agent 预设

### 权限与治理机制
- **AGENTS.md capability manifest**（go generate 生成）：HTTP/WS/proxy mediation、MCP proxy scanning、Signed action receipts 均 Free；`agents.<profile>` 为 Pro（license gate）
- **CHARTER.md**：founder-led 治理承诺；License Promise（core 永 Apache-2.0）；GOVERNANCE.md / SECURITY.md / CODE_OF_CONDUCT.md / CONTRIBUTING.md
- docs/security/：threat models（conductor/audit-packet/per-deployment-ca）、tier-gating-audit-matrix、key-rotation-runbook、monotonic-state-threat-boundary、self-update-signing-custody
- enterprise/：ELv2 付费代码 + license-service + edition gate（cmd enterprise.go）
- 签名基础设施：signing/、anchor/、rules keyring、SPIFFE SVID（svid/svidsidecar）、coveragecert

### 外部依赖（go.mod 直接依赖，突出项）
- common-fate/httpsig（HTTP 签名）、go-jose/go-jose/v4（JOSE）、landlock-lsm/go-landlock（沙箱）、spiffe/go-spiffe/v2（SVID）、fsnotify、sentry-go、prometheus client、rs/zerolog、spf13/cobra、santhosh-tekuri/jsonschema、dunglas/httpsfv、gobwas/ws、goccy/go-yaml、google/uuid、OTLP proto

## 考古焦点（阶段④将验证）
1. mediator-signed action receipts 全链：决策→sanitize→sign→chain/group→emit→verify（Evidence/Authority 流）
2. airlock 代理主循环与 fail-closed 路径（guard/apply_linux、failclosed_linux_test）
3. 扫描器对四类攻击的检测管线（scanner/shield/decide）
4. MCP/A2A 协议中介与 schema canonical 约束
5. 规则签名链更新治理（rules/verify/freshness/transaction）
6. 与 corpus 对照：guardian/aigis（确定性防护）、RAMPART（外部可信证据）、agent-governance-toolkit（治理层）、owasp-agentic-skills
