# 05 Candidates · 未验证假说与开放问题

> 严格区分于 KO：以下内容**不写成已验证知识**。每条标注类型、当前证据、缺失证据、验证路径。

---

## C-01 `checkInitialized` 被注释是"临时测试妥协"还是"长期 fail-open"

- **类型**: 不确定结论（动机不可从仓库证实）
- **当前证据**: `server.go:1730-1743` 注释写明 "Right now it is flaky, because server tests don't await the initialized notification"，并留 TODO；函数恒返回 nil（E-30）。
- **缺失证据**: 相关 issue/PR 讨论、是否计划在 v1.9.0 恢复；测试是否已改为等待 initialized。
- **验证路径**: 查 git history（本轮浅克隆无历史）、issue tracker、`nightly.yml` 是否跑严格模式。
- **影响**: 若长期保持，旧协议客户端可在未 initialized 时调用工具；这是安全/兼容的真实缺口。

## C-02 许可证过渡的文件头未同步是否会引发合规误判

- **类型**: 观察（事实清楚，后果未验证）
- **当前证据**: LICENSE 声明 MIT→Apache-2.0 过渡；120 个 Go 文件头写 "MIT-style license"，0 个写 "Apache License"；GitHub API `NOASSERTION`（E-51/E-52）。
- **缺失证据**: 官方对文件头的迁移计划；SPDX 扫描工具在该仓库的实际输出。
- **验证路径**: 用 `licensee`/`scancode` 跑一次，比较结果与 LICENSE 声明。
- **影响**: 下游企业合规扫描可能把项目判为 MIT（或混合），与项目意图不一致。

## C-03 无状态模式下 server→client notifications 的实际可达性

- **类型**: 暂定模式（文档声明，未运行验证）
- **当前证据**: `StreamableHTTPOptions.Stateless` 注释称"通知可在请求上下文内送达，服务器主动请求立即拒绝"（E-09）。
- **缺失证据**: 实际 HTTP 响应中通知是否真能被客户端收到（取决于响应是否已提交、是否 SSE）。
- **验证路径**: 构造一个在 handler 内发 `notifications/message` 的无状态服务端，用 `StreamableClientTransport` 观察。
- **影响**: 若通知在 JSON 响应模式下被丢弃，"无状态 + 通知"组合能力被高估。

## C-04 客户端如何处理 409 `stream ID conflicts`

- **类型**: 不确定结论
- **当前证据**: 服务端对并发 replay 返回 409（E-38）。
- **缺失证据**: `StreamableClientTransport` 对 409 的处理分支（是否重试、是否当作会话不存在）。
- **验证路径**: 读 `streamable.go` 客户端侧错误处理；构造并发重连测试。
- **影响**: 若客户端把 409 当致命错误，重连会失败而非退避。

## C-05 覆盖率的分子/分母口径

- **类型**: 方法论待明确
- **当前证据**: 本包覆盖 `mcp/` 核心（shared/streamable/server/client/protocol/transport/event）与 `auth/auth.go`；`auth/authorization_code.go`、`auth/extauth/*`、`oauthex/*`、`internal/jsonrpc2/conn.go` 仅做结构阅读，未逐行提取 EK。
- **缺失证据**: 以"文件数/行数/EK 覆盖的符号数"三种口径分别计算覆盖率。
- **验证路径**: 用 `go list` + `grep` 生成符号清单，与本包 EK/Flow 的 symbol 集合求交。
- **影响**: 若不写口径，"覆盖率"数字不可比；本轮不宣称"全覆盖"。

## C-06 跨项目假说：discover-first 是否适用于其他协议迁移

- **类型**: 跨项目假说（`Cross-project validation pending`）
- **当前证据**: KO-01（本项目的探测-回退双栈）。
- **缺失证据**: 其他项目（如 HTTP/2→HTTP/3、GraphQL 订阅迁移）是否也采用"探测-回退"而非"版本开关"。
- **验证路径**: 在 corpus 中比较 2~3 个协议迁移项目，检查是否存在同类结构。
- **影响**: 若成立可提升为 L4；当前只能写"go-sdk 中验证的实践"。

## C-07 HEAD 实际面向 v1.9.0，但 Release 只到 v1.8.0

- **类型**: 版本判定不确定
- **当前证据**: GitHub Release 最新 v1.8.0（2026-09-14）；`docs/mcpgodebug.md` 已存在 `### 1.9.0` 段落（E-50）；HEAD 提交 2026-11-06。
- **缺失证据**: `version.go` 或 tag 对象（浅克隆无 tag）。
- **验证路径**: 查 `git ls-remote --tags` 或 Release 页面。
- **影响**: 本包以 commit 为准，不把 HEAD 标成 v1.8.0 或 v1.9.0；metadata 记 `version: v1.8.0 (latest release) / HEAD post-v1.8.0`。
