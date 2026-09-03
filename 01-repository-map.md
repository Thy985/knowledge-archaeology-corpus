# Codex 仓库地图（Repository Map）

> 由 repository-mapper 角色产出。按知识密度标记 A/B/C 级路径。
> 项目：openai/codex | 类型：Agent 系统 / CLI 工具 / 运行时 | 语言：Rust（112 crates）

---

## 一、项目概览

**一句话定位**：Codex CLI 是 OpenAI 出品的本地运行编程 Agent，通过 Rust 实现的 Agent 运行时，连接 LLM、工具执行、沙箱、MCP 协议，在终端中提供自主编程能力。

**核心技术栈**：
- 语言：Rust（112 crates，workspace 管理）
- 构建：Bazel + Cargo 双构建系统
- UI：ratatui（TUI）
- 协议：JSON-RPC（app-server）、MCP（Model Context Protocol）
- 沙箱：Linux（bwrap/seccomp）、macOS（Seatbelt）、Windows（自定义 sandbox service）
- SDK：Python / TypeScript

---

## 二、A 级路径（必须读，高知识密度）

### 2.1 核心 Agent 运行时
| 路径 | 知识密度 | 关注 |
|------|---------|------|
| `codex-rs/core/src/` | ★★★★★ | 472 个文件，Agent 核心逻辑：session/turn/tool dispatch/context/compact/guardian/multi-agent/exec policy |
| `codex-rs/core/src/session/` | ★★★★★ | Session 生命周期、turn 管理、step activation、token budget |
| `codex-rs/core/src/tools/` | ★★★★★ | 工具系统：registry/router/orchestrator/handlers（shell/apply_patch/mcp/multi_agents/plan） |
| `codex-rs/core/src/agent/` | ★★★★☆ | 多 Agent：control/registry/role/residency/spawn/user_authorization |
| `codex-rs/core/src/guardian/` | ★★★★☆ | Guardian 审批系统：approval_request/review/review_session/prompt/feedback |
| `codex-rs/core/src/context/` | ★★★★☆ | 上下文片段系统：50+ 种 ContextualUserFragment，world_state 渲染 |
| `codex-rs/core/src/exec_policy/` | ★★★★☆ | 执行策略：model_policy/executable_identity/权限配置 |
| `codex-rs/core/src/unified_exec/` | ★★★★☆ | 统一执行：process/process_manager/process_state/stdin_approval/shell_snapshot |

### 2.2 执行与沙箱
| 路径 | 知识密度 | 关注 |
|------|---------|------|
| `codex-rs/exec/` | ★★★★★ | 执行环境抽象 |
| `codex-rs/exec-server/` | ★★★★★ | 独立执行服务器（跨 OS 远程执行） |
| `codex-rs/sandboxing/` | ★★★★★ | 沙箱抽象层 |
| `codex-rs/linux-sandbox/` | ★★★★☆ | Linux bwrap/seccomp 沙箱实现 |
| `codex-rs/windows-sandbox-rs/` | ★★★★☆ | Windows 沙箱 Rust 端 |
| `codex-rs/windows-sandbox-service/` | ★★★☆☆ | Windows 沙箱服务（C#?） |
| `codex-rs/bwrap/` | ★★★☆☆ | bwrap 绑定 |

### 2.3 工具与协议
| 路径 | 知识密度 | 关注 |
|------|---------|------|
| `codex-rs/tools/` | ★★★★☆ | 工具定义（独立 crate） |
| `codex-rs/mcp-server/` | ★★★★★ | MCP 服务器实现 |
| `codex-rs/codex-mcp/` | ★★★★☆ | MCP 连接管理（mcp_connection_manager.rs） |
| `codex-rs/rmcp-client/` | ★★★☆☆ | Remote MCP 客户端 |
| `codex-rs/app-server/` | ★★★★★ | JSON-RPC API 服务器（IDE/桌面端集成） |
| `codex-rs/app-server-protocol/` | ★★★★☆ | API 协议定义（v1/v2） |
| `codex-rs/protocol/` | ★★★★☆ | 核心协议类型 |

### 2.4 模型与配置
| 路径 | 知识密度 | 关注 |
|------|---------|------|
| `codex-rs/model-provider/` | ★★★★★ | 模型提供抽象（OpenAI/兼容 API） |
| `codex-rs/models-manager/` | ★★★☆☆ | 模型管理 |
| `codex-rs/config/` | ★★★★☆ | 配置系统（独立 crate） |
| `codex-rs/config-schema/` | ★★★☆☆ | 配置 schema 生成 |

### 2.5 记忆与状态
| 路径 | 知识密度 | 关注 |
|------|---------|------|
| `codex-rs/memories/` | ★★★★☆ | 记忆系统 |
| `codex-rs/history/` | ★★★★☆ | 历史记录 |
| `codex-rs/message-history/` | ★★★☆☆ | 消息历史 |
| `codex-rs/state/` | ★★★★☆ | 状态管理 |
| `codex-rs/thread-store/` | ★★★★☆ | 线程存储 |
| `codex-rs/thread-manager-sample/` | ★★☆☆☆ | 示例 |

### 2.6 治理文档
| 路径 | 知识密度 | 关注 |
|------|---------|------|
| `AGENTS.md` | ★★★★★ | Agent 治理文档：编码规范、core crate 反膨胀、模型上下文规则、测试规范、API 设计、变更大小限制 |
| `docs/` | ★★★★☆ | 16 个文档：config/exec/execpolicy/sandbox/skills/authentication/slash_commands |

---

## 三、B 级路径（值得扫，中等知识密度）

| 路径 | 关注 |
|------|------|
| `codex-rs/cli/` | CLI 入口与命令解析 |
| `codex-rs/tui/` | TUI 界面（ratatui），snapshot 测试 |
| `codex-rs/prompts/` | Prompt 模板 |
| `codex-rs/plugin/` | 插件系统 |
| `codex-rs/core-plugins/` | 核心插件 |
| `codex-rs/skills/` | 技能系统 |
| `codex-rs/hooks/` | Hook 系统 |
| `codex-rs/rollout/` | 灰度发布 |
| `codex-rs/features/` | 特性开关 |
| `codex-rs/agent-roles/` | Agent 角色定义 |
| `codex-rs/agent-identity/` | Agent 身份 |
| `codex-rs/guardian-context/` | Guardian 上下文 |
| `codex-rs/context-fragments/` | 上下文片段 |
| `codex-rs/apply-patch/` | 补丁应用（独立 crate） |
| `codex-rs/shell-command/` | Shell 命令抽象 |
| `codex-rs/file-system/` | 文件系统抽象 |
| `codex-rs/git-utils/` | Git 工具 |
| `codex-rs/file-search/` | 文件搜索 |
| `codex-rs/file-watcher/` | 文件监听 |
| `codex-rs/secrets/` | 密钥管理 |
| `codex-rs/keyring-store/` | Keyring 存储 |
| `codex-rs/login/` | 登录认证 |
| `codex-rs/chatgpt/` | ChatGPT 集成 |
| `codex-rs/backend-client/` | 后端客户端 |
| `codex-rs/codex-client/` | Codex 客户端 |
| `codex-rs/codex-api/` | Codex API |
| `codex-rs/responses-api-proxy/` | Responses API 代理 |
| `codex-rs/http-client/` | HTTP 客户端 |
| `codex-rs/network-proxy/` | 网络代理 |
| `codex-rs/ollama/` | Ollama 集成 |
| `codex-rs/lmstudio/` | LM Studio 集成 |
| `codex-rs/voice-host/` | 语音宿主 |
| `codex-rs/realtime-webrtc/` | Realtime WebRTC |
| `sdk/python/` | Python SDK |
| `sdk/typescript/` | TypeScript SDK |
| `codex-cli/` | CLI 包（npm 分发） |
| `scripts/` | 构建/测试脚本 |
| `justfile` | 任务定义 |
| `.github/workflows/` | CI/CD |

---

## 四、C 级路径（可跳过，低知识密度）

| 路径 | 原因 |
|------|------|
| `bazel/` | 构建系统配置 |
| `patches/` | 第三方依赖补丁（v8/webrtc/zstd 等） |
| `third_party/` | 第三方依赖 |
| `tools/argument-comment-lint/` | Lint 工具 |
| `environments/` | 环境配置 |
| `.devcontainer/` | 开发容器 |
| `.vscode/` | IDE 配置 |
| `codex-rs/vendor/` | vendored 依赖 |
| `codex-rs/v8-poc/` | V8 概念验证 |
| `codex-rs/analytics/` | 分析遥测 |
| `codex-rs/otel/` | OpenTelemetry |
| `codex-rs/async-utils/` | 异步工具 |
| `codex-rs/utils/` | 通用工具 |
| `codex-rs/ansi-escape/` | ANSI 转义 |
| `codex-rs/arg0/` | arg0 处理 |
| `codex-rs/build-info/` | 构建信息 |
| `codex-rs/clippy.toml` | Lint 配置 |
| `codex-rs/deny.toml` | 依赖审计配置 |

---

## 五、知识藏区标注（高知识密度预判）

1. **沙箱与执行边界**：`sandboxing/` + `linux-sandbox/` + `windows-sandbox-rs/` + `exec-server/` + `unified_exec/` —— Agent 系统最核心的安全边界
2. **Guardian 审批**：`core/src/guardian/` + `guardian-context/` —— 人类审批循环的实现
3. **多 Agent 控制**：`core/src/agent/control/` + `agent-roles/` + `agent-identity/` —— 多 Agent 协作与权限
4. **工具调度**：`core/src/tools/orchestrator.rs` + `router.rs` + `registry.rs` —— 工具调用的核心调度
5. **上下文工程**：`core/src/context/` + `context-fragments/` + `compact/` —— 50+ 上下文片段的组装与压缩
6. **执行策略**：`core/src/exec_policy/` + `config/permissions.rs` —— 什么能执行、什么需要审批
7. **MCP 协议**：`mcp-server/` + `codex-mcp/` —— 工具扩展协议
8. **AGENTS.md 治理规则**：core 反膨胀、上下文硬限制、变更 800 行限制 —— 工程决策的集中体现

---

## 六、Wave 1 Discovery 角色分配

| 角色 | 聚焦路径 |
|------|---------|
| code-analyst | `core/src/session/`、`core/src/tools/`、`core/src/agent/`、`core/src/guardian/`、`unified_exec/`、`sandboxing/`、`mcp-server/`、`app-server/` |
| doc-analyst | `AGENTS.md`、`docs/config.md`、`docs/exec.md`、`docs/execpolicy.md`、`docs/sandbox.md`、`docs/skills.md`、`docs/authentication.md`、`docs/slash_commands.md` |
| test-analyst | `core/src/session/tests/`、`core/suite/`（集成测试）、`core-test-support/`、各 crate `*_tests.rs` |
| failure-analyst | git log（bugfix/regression）、`patches/`、`TODO/FIXME`、fallback/retry 路径 |
| authority-analyst | `core/src/exec_policy/`、`core/src/guardian/`、`core/src/config/permissions.rs`、`sandboxing/`、`linux-sandbox/`、`windows-sandbox-rs/`、`core/src/tools/approvals.rs`、`core/src/agent/control/user_authorization.rs` |
