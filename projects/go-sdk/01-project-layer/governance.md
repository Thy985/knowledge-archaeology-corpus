# 01.5 Governance · 权限、授权与仓库治理

## 1. 运行时权限边界

| 层 | 机制 | 位置 |
|---|---|---|
| 传输入口 | DNS rebinding 防护（loopback 服务拒绝非 loopback Host） | `streamable.go:385-397` |
| 传输入口 | `CrossOriginProtection`（可选） | `streamable.go:400-406` |
| 传输入口 | 请求体上限（413） | `streamable.go:409-412` |
| 授权 | `auth.RequireBearerToken(verifier, opts)` 中间件 | `auth/auth.go:97` |
| 令牌 | `TokenInfo{UserID, Scopes, Expiration}` 注入 context | `auth/auth.go:21/83` |
| 会话 | `lookupSession` 比对 `TokenInfo.UserID`，不匹配 403 | `streamable.go:642-660` |
| 协议 | ≥2026-07-28 禁止 server-initiated sampling/roots/elicitation | `server.go:1758` |
| OAuth | authorization code、client credentials、OIDC login、enterprise handler | `auth/`、`auth/extauth/` |
| OAuth 元数据 | Protected Resource Metadata（RFC 9728）、DCR、token exchange | `oauthex/` |

## 2. 授权检查顺序（`auth/auth.go:133-196`）

1. 解析 `Authorization: Bearer <token>`；缺失 → 401（且不带 `invalid_token`，符合 RFC 6750 §3.1）。
2. 调用 `TokenVerifier`；`ErrInvalidToken` → 401 `invalid_token`；`ErrOAuth` → 400；其他 → 500。
3. 检查过期（**先于 scope**）：
   - 无 expiration → 默认 401（`AllowMissingExpiration` 默认 false）；
   - 过期（含 `ClockSkew`）→ 401 `invalid_token`。
4. 检查 scope：全部命中才通过；缺失 → 403 `insufficient_scope`。

> 设计意图明确写在注释里：过期令牌返回 401 让客户端换新令牌；scope 不足返回 403 让客户端申请更多 scope。两者**不能混用**。

## 3. 会话与身份绑定

- 有状态会话创建时复制 `TokenInfo.UserID` 到 `sessionInfo.userID`。
- 后续任何 GET/POST/DELETE 都经 `lookupSession`；若 `userID != ""` 且 token 用户不匹配 → 403 `session user mismatch`。
- **无状态模式不执行此检查**（没有跨请求会话可绑定）——身份校验退化为"每个请求的 token 校验"，而不是"会话绑定"。

## 4. 协议治理

| 规则 | 证据 |
|---|---|
| 版本只能收窄不能拓宽 | `ServerOptions.SupportedProtocolVersions` 注释 |
| 旧握手永不拒绝 | 同上；无可用版本时返回 2025-11-25 让客户端断开 |
| 新协议 server-initiated 请求禁止 | `assertServerInitiatedRequestAllowed` |
| 弃用兼容 ≥12 个月 | README（SEP-2577） |
| 旧 SSE 传输保留 | `mcp/sse.go` |
| 无状态兼容开关 | `MCPGODEBUG=allowsessionsinstateless=1` |

## 5. 仓库治理

- **AGENTS.md**：构建/测试命令、gofmt、版权头、注释只写 why、文档由 weave 生成。
- **CONTRIBUTING.md**：贡献流程（未逐条展开，属文档层）。
- **SECURITY.md**：安全披露渠道。
- **CI**：8 个 workflow；`require-linked-issue.yml` 要求 PR 关联 issue；`scorecard.yml` 跑 OpenSSF Scorecard。
- **许可证**：MIT → Apache-2.0 过渡；文档 CC-BY-4.0；未获重许可同意的 MIT 贡献仍为 MIT。

## 6. 治理判断

1. **运行时授权是"中间件 + 会话绑定"两层**：中间件保证每个请求有合法 token；会话绑定保证 token 不能在会话中途被换人。
2. **无状态化削弱了会话绑定**：这是"水平扩展"与"会话级防劫持"之间的真实取舍，不是实现遗漏。
3. **协议治理是"单向兼容"**：新版可以拒绝旧版（返回支持列表），旧版握手不能拒绝客户端，只能降级并让客户端自行断开。
4. **许可证处于混合态**：LICENSE 声明过渡，但 120 个 Go 文件头仍写 "MIT-style license"，0 个写 Apache；这是文档/合规层的一个未完成迁移（见 C-02）。
