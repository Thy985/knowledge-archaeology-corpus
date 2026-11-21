# Independent Validation Report（盲重建）— ARCH-2026-10-16-001 · Pipelock

> Auditor 独立重读仓库（不引用考古 Package 00-06 为事实来源），建立 Independent Findings 后与考古产物对比。禁止修改原考古产物；本报告为修正输入（阶段⑥ Reconciliation 用）。

## 判定统计
| 判定 | 数量 | 项 |
|---|---|---|
| CONFIRMED | 9 | 签名链（EK-01/02/04）、Unmarshal 三面（EK-03）、airlock fail-closed（EK-15）、guard 拒绝启动（EK-24）、MCP parse 默认 block（EK-21）、rules reload 拒绝（EK-33）、response_incomplete 全局（EK-28）、"签名≠决策正确"标注（EK-08）、C2（决策签名≠内容签名） |
| PARTIALLY_CONFIRMED | 1 | KO-01（本仓库内成立；跨项目证据仍 pending——维持 Validated Pattern 不升 Principle） |
| DOWNGRADED | 0 | — |
| OVER_GENERALIZED | 0 | — |
| MISSING | 3 | kill switch 七激活源细节；browser_cache 也触发 response_incomplete；OnParseError 默认 ActionBlock 证据 |
| CONTRADICTED | 0 | — |
| NEEDS_HUMAN_REVIEW | 2 | C6（enterprise/fleet 权威汇聚，ELv2 未读）；A5（SPIFFE posture 未深读，原 04 标 PASS 过强） |

## 3 个成功（强声明被独立验证）
1. **外部信任锚强制是真实而非名义**：`cmd/pipelock-verifier/receipt.go:140-160` —— 无 pinned key 时走 VerifyInternalConsistencyOnly + `report.Unpinned=true` + **`report.Valid=opts.allowUnpinned`（默认 false）** → exit General + "verify: unpinned receipt"。验证器对无锚收据默认判 invalid。EK-02 从"Verify() 故意不可用"扩展到"无锚路径默认无效"。
2. **fail-closed 是默认配置而非仅文档声称**：`internal/config/normalize.go:371-374` —— "Always default OnParseError (fail-closed) regardless of enabled state" → ActionBlock。EK-21 获得默认值证据。
3. **规则包降级需要显式 opt-in**：`internal/cli/runtime/server_reload.go:575` —— "accepting degraded rule-bundle coverage after explicit operator opt-in"（rules.allow_degraded）。EK-33 的"strict 拒绝 + 非 strict 事件化"获得实现级确认。

## 3 个错误/需修正（独立 Auditor 对考古产物的修正输入）
1. **ERR-1（补充，非推翻）**：06/02 中 EK-21 的表述"despite on_parse_error: forward"缺默认值；独立证据：OnParseError 默认即 ActionBlock（normalize.go:373-374）。修正：EK-21 补"默认 fail-closed + 显式 forward 亦拒结构性问题"双证据。
2. **ERR-2（遗漏）**：EK-28 未覆盖 browser_cache 触发路径：`internal/proxy/response_conditions.go:40`（`replaceWithBlockReason(..., ResponseIncomplete, "browser_cache")`）。response_incomplete 不只在 forward/intercept，还在 cache-validator 条件路径。
3. **ERR-3（遗漏）**：kill switch 激活源"Seven activation sources (config, API, Conductor remote kill, Conductor stale…)"（killswitch.go:5-7）未入 04 P3。补充：kill switch 有 ≥7 激活源 + EnvAPIToken 覆盖 + 独立端口豁免设计。

## 遗漏清单（MISSING 展开）
- M-1：kill switch 激活源明细（4/04 P3 补全）
- M-2：browser_cache → response_incomplete（EK-28 补全）
- M-3：MCP OnParseError 默认 ActionBlock（EK-21 补全）
- M-4：`plain CONNECT hostname-only projection`（forward.go:215）——README 声明的扫描面缩小的代码级证据（C2 补强）

## Benchmark case（端到端对照）
- **Case 1（本仓库）**：无 pinned key 验证器对签名收据 → `unpinned receipt` banner + Valid=false（默认）→ 外部锚缺失即证据无效。与 RAMPART"观测即契约"对照：Pipelock 把"信任锚"做成验证器的硬前置。
- **Case 2（跨仓库）**：guard 的 ErrManifestIncomplete（拒绝启动）vs guardian 的 fail-closed deny vs ai-protector 默认拒绝——"拒绝而非收窄"三实例独立确认（KO-02 跨项目 pending 依据）。
- **Case 3（攻击面）**：C2（决策签名≠内容签名）+ plain CONNECT hostname-only → 中介签名的真实覆盖范围 = "决策"而非"内容"——KO-01 的解释范围必须包含此边界声明。

## 结论
独立盲重建未发现 CONTRADICTED 或 OVER_GENERALIZED；1 项 PARTIALLY（KO-01 跨项目），3 项 MISSING 为补充性修正，2 项 NEEDS_HUMAN_REVIEW（enterprise/ ELv2 边界）。考古 Package 主干 CONFIRMED。修正项随阶段⑥ Reconciliation 并入 corpus 最终版（不修改原考古产物文件，修正记录于本文件与 corpus 06_validation）。
