# 01 · Project Layer — Omnigent（项目地图）

> 本层为 L0 工程事实，全部可追溯到仓库实际内容（commit 381bf63）。

## 1.1 项目定位

开源 meta-harness：统一编排层 over 既有 AI coding agents（README.md 首段）。三支柱（README "Why Omnigent?"）：**Composition**（组合/互换 harness）、**Control**（策略、沙箱、花销上限——作用于 server/agent/chat 三级）、**Collaboration**（实时共享会话、跨设备）。

## 1.2 架构（模块级）

```
┌────────────────────────── 控制面（server/）──────────────────────────┐
│  FastAPI app（app.py）· auth.py · feature_flags.py · bundles.py       │
│  managed_hosts.py（云沙箱供应）· API.md / DBSPEC.md · accounts_*     │
│  stores/（conversation_store / host_store / … SQLAlchemy）           │
└──────────────┬───────────────────────────────────────────────┬────────┘
               │ REST/SSE                                       │ launch token
┌──────────────▼──────────────────┐                ┌────────────▼─────────┐
│  runner/（会话运行时）            │                │ managed sandbox host │
│  app.py·orchestration.py        │                │（云沙箱内 omnigent   │
│  tool_dispatch.py·policy.py     │                │  host 进程）          │
│  pending_approvals.py·mcp_      │                └──────────────────────┘
│  manager.py·identity.py         │
└──────┬─────────────────┬────────┘
       │ native bridge   │ tmux/file-inject/stdio
┌──────▼─────────────────▼──────────────────────────────────────────────┐
│  per-harness 适配（claude_native*.py / codex_native*.py / …）         │
│  11 个 harness：claude/codex/cursor/hermes/pi/kimi/kiro/qwen/goose/  │
│                 opencode/antigravity                                  │
└────────────────────────────────────────────────────────────────────────┘
```

## 1.3 核心模块

| 模块 | 职责 | 证据 |
|------|------|------|
| runner/ | agent 会话运行时：turn 路由、工具分发、策略执行、审批队列、子 agent 路由、MCP 管理 | runner/app.py（12.7k 行）、runner/tool_dispatch.py |
| server/ | 多租户控制面：认证、feature flags、managed hosts、bundles、API schema | server/app.py、server/schemas.py（4.9k 行） |
| policies/ | 声明式策略系统：PolicySpec → Policy 求值器 → registry | policies/base.py、policies/registry.py、policies/builtins/（13 个模块） |
| sandbox/ | 本地沙箱（bwrap/seatbelt） | sandbox/bwrap.py、sandbox/seatbelt.py |
| *native_bridge* | per-harness 原生适配 | claude_native_bridge.py 等 11 组 + native_bridge_common.py |
| stores/ | 会话/消息/状态持久化 | stores/conversation_store/sqlalchemy_store.py（4.4k 行） |
| spec/ | Agent YAML 声明式 spec（AgentSpec/PolicySpec/ExecutorSpec） | spec/parser.py（3.8k）、spec/types.py |
| inner/ | 内部执行器（codex/claude_sdk/pi）与 OS 环境 | inner/codex_executor.py 等 |
| llms/ + model_catalog* | 模型目录、解析、降级 | model_resolver.py、model_fallbacks.py |
| runtime/ | 核心 agent loop（workflow.py）+ policy engine + telemetry | runtime/workflow.py、runtime/policies/engine.py |

## 1.4 生命周期

- **会话生命周期**：session_lifecycle.py、suspend_watch.py、resume_dispatch.py、session_import/（导入导出）
- **runner 生命周期**：runner/_entry.py（启动）、runner/_zygote.py、_runner_startup.py、_runner_startup_profile.py
- **沙箱生命周期**：managed_hosts.py（provision → host register → relaunch 原地覆盖）；reaper 离线回收（默认 30 天）
- **内置 agent 资产**：bundles.py（artifact store；缺失时 server 启动自愈重传）

## 1.5 主要状态

- 每 harness 会话状态文件（claude_native_state.py / codex_native_state.py / cursor_native_status.py / opencode_native_state.py）
- server/feature_flags.py（特性开关）
- stores/（SQLAlchemy：SqlConversation / SqlConversationItem / SqlPolicy / SqlSessionPermission / SqlUserDailyCost 等，db/db_models.py）

## 1.6 配置与入口

| 项 | 值 | 证据 |
|----|-----|------|
| Python 包 | omnigent 0.13.0.dev0，requires-python >=3.12 | pyproject.toml |
| CLI 入口 | `omnigent`（__main__.py → cli.py 12.7k 行） | omnigent/__main__.py |
| Server 入口 | `omnigent server`（FastAPI app） | server/app.py |
| 安装 | scripts/install_oss.sh（curl 一键） | README |
| 本地开发 | justfile（just dev / just lint） | AGENTS.md |
| 部署 | railway.toml / render.yaml / deploy/ | 仓库根 |

## 1.7 权限 / policy / governance 机制

- policies/（PolicySpec 声明式；作用域 server/agent/chat 三级——README）
- runner/policy.py + runner/pending_approvals.py（runner 侧执行 + 审批）
- sandbox/（bwrap/seatbelt 进程隔离）
- per-harness permissions 文件（cursor_native_permissions.py / hermes_native_permissions.py / opencode_native_permissions.py / qwen_native_permissions.py / kiro_native_permissions.py / goose_native_permissions.py）
- server/auth.py + accounts_*（认证/账户）；git_credential_github.py
- 仓库治理：AGENTS.md / CLAUDE.md / CONTRIBUTING.md / DCO / designs/ci-external-contributors-proposal.md / designs/contributor-review-merge-proposal.md

## 1.8 外部依赖

- 云沙箱后端：Modal/Daytona/Blaxel/Islo/E2B/CoreWeave/K8s/OpenShell/Boxlite/microsandbox/Databricks（managed_hosts.py provider 枚举）
- vendor harness CLI：11 个（见 1.2）
- SQLAlchemy（存储）、FastAPI（server）、bubblewrap（Linux 沙箱）、Seatbelt（macOS 沙箱）
- pnpm monorepo（web/ + sdks/）、Electron（desktop）

## 1.9 显著设计文档入口（ADR 证据）

designs/CLI_CONTRACT.md · OBSERVABILITY.md · harness-plugin-interface.md · harness-modular-registry-proposal.md · CREDENTIAL_STORE.md · FEATURE_FLAGS.md · CUJ-ANALYSIS/CUJ-MAP.md · DEVICE_AUTH.md · docs/AGENT_YAML_SPEC.md · docs/harness-bench-design.md · docs/DEVIN_ACP_DESIGN.md
