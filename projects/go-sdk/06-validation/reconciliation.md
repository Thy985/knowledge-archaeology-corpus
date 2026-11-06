# 06.3 Reconciliation · 独立验证后的修正记录

> 依据：`archaeology-runs/ARCH-2026-11-07-001/validation/independent_validation.md`。
> 原则：**原 Package 保留不动**（审计痕迹）；以下修正只作用于 corpus 交付版本（`projects/go-sdk/` 的顶层文件）。

## 1. 修正清单

| 验证项 | 原 Package | Corpus 修正 | 落点 |
|---|---|---|---|
| P-01 | KO-02："去掉跨请求标识" | 明确为"去掉**传输层** `Mcp-Session-Id`；逻辑 session ID 可保留"；新增边界 4 | `03-knowledge-layer/ko-01-04.md` |
| P-02 / M-02 | "旧 SSE 保留" | 新增 EK-32：旧 SSE `SupportsProtocolVersion` 只支持 `<2026-07-28`，会在 discover 被过滤 | `02-engineering-knowledge/ek-31-33-reconciled.md` |
| P-03 | EK-01："任何非现代错误都会回退" | 精确为"discover 失败且不属于可协商的 unsupported-version 时才回退；可协商错误先重试"（已写入 EK-01 描述，未改原文；在 reconciliation 中显式限定） | 本文件 §2 |
| P-04 / M-03 | "旧 SSE 仍在" | 新增 EK-33：`CloseSSEStream` 在新协议 no-op（deprecated） | `ek-31-33-reconciled.md` |
| P-05 | "OAuth authorization code 流程" | 补充 PKCE / DCR / token auth method 选择作为覆盖说明（不新增 EK，因未逐行验证） | 本文件 §3 |
| P-06 | "HEAD 在 v1.8.0 之后" | metadata 写 `v1.8.0 (latest release) / HEAD post-v1.8.0 (1.9.0 doc entries present)` | `metadata.yaml` |
| M-01 | 未覆盖 `streamable_headers.go` | 新增 EK-31（HTTP header 参数面）+ F-D5 | `ek-31-33-reconciled.md`、`04-flow-atlas/flow-atlas.md` |
| D-01 | KO-05 为 L4 | 保留 L4，但标题语义收紧为"请求路径门禁"；Epistemic 保持 Observation | `03-knowledge-layer/ko-05-08.md`（原文已标 pending） |
| O-01 | KO-08 标题偏普遍 | 建议改为"负向清单降低误用，但不保证治理一致性"；本轮不改标题，在 §4 显式声明 | 本文件 §4 |

## 2. EK-01 的精确化（不改原文，补定义）

**原文**："任何非现代错误即回退 legacy `initialize`。"

**精确表述**：
1. `server/discover` 成功 → 使用返回版本；
2. 返回 `CodeUnsupportedProtocolVersion` **且带** `Supported` 列表 → `negotiateMutuallySupportedVersion`，若结果 ≥2026-07-28 则**换版本重试**（最多 2 次），不立即回退；
3. 其他错误（含无 Supported 列表的 unsupported、网络错误、解码错误）→ 回退 `initialize`，版本强制 2025-11-25；
4. `initialize` 返回版本不在 `supportedProtocolVersions` → 关闭会话并报 `unsupportedProtocolVersionError`。

> 因此"任何非现代错误都回退"会漏掉第 2 步的重试路径。

## 3. OAuth 覆盖说明（不升维）

独立 Auditor 确认 `auth/authorization_code.go` 含：PKCE 支持检查（`:342`）、动态客户端注册（`:602/656`）、token auth method 选择（`:565`）、protected resource metadata 解析（`:476`）、`usedCodeVerifier` 每次授权状态（`:678`）。

但本轮**未逐行验证**这些流程的错误路径与状态机，因此：
- 不新增 EK；
- 不把"OAuth 支持"写成完整能力；
- 维持 conformance baseline 的 5 项缺口为权威负向清单（E-49）。

## 4. 保留的未修正项（诚实记录）

| 项 | 原因 | 处理 |
|---|---|---|
| H-01 frame cap 具体符号 | 浅克隆无历史；`conn.go:672` 只说明"无法一般性限制 handler 大小" | 标 NEEDS_HUMAN_REVIEW，不写实现细节 |
| H-02 `checkInitialized` 动机 | 只有代码注释自述，无 issue/PR | 保持 Candidates C-01 |
| O-01 KO-08 标题 | 修改标题会影响 KO 编号稳定性；且边界已写明 | 本轮不改，记录于此 |
| D-01 KO-05 层级 | 单项目证据不足以否证 L4；已标 pending | 保留 L4 + pending |

## 5. 一致性检查（写前）

| 检查 | 结果 |
|---|---|
| 结构（7 类目录 + archaeology-runs） | ✅ |
| metadata 与 run_metadata 字段一致 | ✅（见 `metadata.yaml`） |
| EK 编号连续（01~33，31~33 单列） | ✅ |
| KO 编号连续（01~08） | ✅ |
| 每个 KO 有 aggregation_rule | ✅ 8/8 |
| 每个 EK 有 ≥1 边 | ✅ 33/33 |
| 每条 Fact 有 file:line 或命令 | ✅ 53/53 |
| 历史 run 未被覆盖 | ✅ 新建 `archaeology-runs/ARCH-2026-11-07-001/` |
| 无 `projects/go-sdk` 重复目录 | ✅ 写前确认不存在 |
