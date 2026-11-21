# 00 · Overview — Pipelock（ARCH-2026-10-16-001）

> 一句话定位：**开源 AI agent 防火墙（Verifiable Egress Control）**——作为网络与工具代理中介 AI-agent 流量，扫描外泄/注入/SSRF/工具投毒，并对每次边界决策产出 **mediator-signed action receipts**（外部可离线验证的证据链）。核心命题："**agent 不能成为自身安全权威——决策证据必须由中介外部签名，relying party 在 agent 信任边界之外验证**"。

## 三条主线（本次考古提炼）
1. **决策证据外部化**：每次中介动作 → typed ActionRecord（8+1 类型 × side-effect × reversibility × authority/taint/contract context）→ Ed25519 签名（SHA-256 canonical）→ flight recorder 链/平行组 → 独立 verifier 用持有公钥离线验证。"签名只覆盖 canonical action record；Ext 永不进签名、永不 consult"；`Verify()` 故意不可用（强制 VerifyWithKey 外部信任锚）。
2. **fail-closed 贯穿全栈**：airlock 未知 tier→block；guard manifest 不完整/策略被收窄/二次应用→拒绝启动；config reload 拒绝安全降级（strict mode + 规则包解析失败拒 reload 防静默削弱覆盖）；MCP 输入 duplicate keys/无效 UTF-8/malformed JSON→拒绝（即便 on_parse_error: forward）；3.7.0 起 receipt open prefix 不算 complete（exit 3）。
3. **检测-决策-执行-证据四层分离**：scanner（detect）→ decide（内容感知策略）→ guard/shield/sandbox（OS 级执行强制，Landlock）→ receipt/emit/anchor（外部可验证证据）——中间层互不信任，任何一层失败走 fail-closed。

## 关键数字（来源见各章节）
- HEAD `8ade100ce6fc48f565e916c12b462e006d699a27` · v3.7.0（2026-10-14）· Go 1.26 · Core Apache-2.0 + enterprise ELv2
- 核心代码：cmd 41 文件/12,474 行 + internal 1189 文件/380,608 行（排除测试/生成）；Go 测试文件 2496
- capability manifest：27 项能力，8 项 free + agents/fleet/assess 等 license_feature（enterprise/fleet_gate.go）
- 收据机制：Receipt v1（schema 版本 1）；AEL 流 256MiB 验证上限；receipt_chains 1~32 平行组；10-min stop budget
- 测试：CI（ci/security/hardening/gauntlet/clusterfuzzlite）+ 2496 Go 测试文件 + agent-egress-bench 独立 corpus（v2.14.0，gauntlet pin commit 可复现）

## 质量指标快照（详见 06）
EK 40 / KO 8 / Candidates 8 / Flow Edge 28 / 反例覆盖 8/8 / 盲重建独立执行 / 判定统计见 06。

## 与 corpus 的连接
- guardian / aigis / ai-protector（确定性防护）→ Pipelock guard fail-closed 的 OS 级强化版本（Landlock + 签名证据）
- RAMPART（"观测即契约"）→ Pipelock 的 mediator-signed receipts 是"外部可信证据"的系统化工程（canonical 规范化 + 链 + 独立 verifier）
- agent-governance-toolkit（Agent OS 策略引擎）→ Pipelock 独立实现 + 完整证据链
- mcp / owasp-agentic-skills / pyrit → 威胁面映射（attacks-blocked.md 带 MITRE ATT&CK）
