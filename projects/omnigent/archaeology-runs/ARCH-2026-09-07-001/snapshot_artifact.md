# Repository Snapshot — ARCH-2026-09-07-001 · omnigent

## 快照元信息
| 项 | 值 |
|----|-----|
| repository | https://github.com/omnigent-ai/omnigent.git |
| commit SHA | 381bf638fb31e6a51990d9dab54ea9ef4b933711（浅克隆 HEAD） |
| branch | main |
| repository version | pyproject `0.13.0.dev0`；最新 release **v0.12.0（2026-09-01）**（CHANGELOG 顶部） |
| analysis timestamp | 2026-09-07（SOP 定时轮） |
| skill version | knowledge-archaeology v3.2（本地生产版） |
| license | Apache 2.0（LICENSE） |
| status | alpha（classifier: Development Status 3 - Alpha） |

## 项目基础地图
```
omnigent/
├── runner/            # Agent 运行时（app.py 12.7k 行；native/orchestration.py 8.6k；tool_dispatch.py 8.3k；policy.py；pending_approvals.py；turn_routing/subagent_routing/mcp_manager/identity/resource_registry）
├── server/            # 控制面/协作面（app.py 3.5k；schemas.py 4.9k；managed_hosts.py 3.6k；auth.py；feature_flags.py；bundles.py；API.md/DBSPEC.md）
├── policies/          # 策略系统（base.py + builtins/ + function.py + registry.py + schema.py + types.py）
├── sandbox/           # 沙箱（bwrap.py + seatbelt.py）
├── stores/            # 会话/记忆存储（conversation_store/sqlalchemy_store.py 4.4k）
├── claude_native*.py / codex_native*.py / cursor_native*.py / hermes_native*.py / pi_native*.py / kimi_native*.py / kiro_native*.py / qwen_native*.py / goose_native*.py / opencode_native*.py / antigravity_native*.py
│                      # per-harness 原生适配（bridge + forwarder + permissions + hook + state）
├── native_bridge_common.py / native_policy_hook.py / native_dispatch.py  # 桥接公共层
├── spec/parser.py    # Agent YAML 声明式 spec 解析（3.8k）
├── inner/            # codex_executor / claude_sdk_executor / pi_executor（内部执行器）
├── runtime/workflow.py  # 工作流（3.0k）
├── llms/ + model_catalog*.py + model_resolver.py + model_fallbacks.py + model_override.py  # 模型路由
├── tools/ terminals/ extensions/ db/ telemetry/ host/ environments/ connections/
├── cli.py (12.7k) chat.py repl/ community/ onboarding/ session_import/
├── api/ resources/ entities/ json_types.py config.py errors.py crash_handler.py
└── tests/             # 1681 个 py 测试文件
```

## 主要语言
Python（核心 760 文件 / ~437k 行）+ TypeScript（web/ + sdks/）+ Electron（desktop）+ pnpm monorepo（pnpm-workspace.yaml）+ uv（uv.lock）

## 主要运行入口
- CLI：`omnigent`（omnigent/__main__.py → cli.py；setup.py/pyproject entry）
- Server：`omnigent/server/app.py`（FastAPI 类，含 API.md 文档）
- Runner：`omnigent/runner/app.py` + `runner/_entry.py` + `runner/native/orchestration.py`
- 安装脚本：scripts/install_oss.sh

## 核心模块与职责
| 模块 | 职责 |
|------|------|
| runner/ | agent 会话运行时：turn 路由、工具分发、审批、策略执行、子 agent 路由、MCP 管理 |
| server/ | 多租户控制面：认证、feature flags、managed hosts（云沙箱供应）、bundles（内置 agent 资产） |
| policies/ | 声明式策略：base/registry/schema/function/builtins（暂停审批、花销上限、工具限制） |
| sandbox/ | 本地沙箱：bwrap（bubblewrap）+ seatbelt（macOS） |
| *native_bridge* | per-harness 适配：把 Claude Code/Codex/Cursor/Hermes/Pi/Kimi/Kiro/Qwen/Goose/OpenCode/Antigravity 包成统一 runner 会话 |
| stores/ | 会话/消息/状态持久化（SQLAlchemy） |
| spec/parser | Agent YAML（声明式 agent 定义）解析 |
| model_catalog/resolver/fallbacks | 模型路由与降级 |

## 核心数据结构
- runner/identity.py（agent 身份）；runner/resource_registry.py（资源注册）；server/schemas.py（API schema 4.9k 行）；spec/parser.py（AgentSpec AST）
- stores/conversation_store/sqlalchemy_store.py（会话/消息 ORM）
- policies/schema.py + types.py（PolicySpec 结构）

## 核心状态
- 会话生命周期（session_lifecycle.py、suspend_watch.py、resume_dispatch.py）
- runner 状态（native/ per-harness state.py 文件：claude_native_state/codex_native_state/cursor_native_status/opencode_native_state）
- 特征开关（server/feature_flags.py）

## 主要测试体系
- tests/ 1681 个 py 文件（单元 + 集成；含 _e2e_policy_callables.py 等策略 e2e）
- pre-commit 全量检查（AGENTS.md）；`just lint` / `just dev`

## 主要配置
- pyproject.toml（Python 包配置 + path-deps SDK）
- AGENTS.md / CLAUDE.md（仓库 agent 指导）
- justfile（开发 recipe）；railway.toml / render.yaml（部署）
- openapi.json（server API）

## 主要权限 / policy / governance 机制
- policies/（声明式策略：审批暂停/花销上限/工具限制，作用域 server/agent/chat 三级）
- runner/policy.py + runner/pending_approvals.py（审批队列）
- sandbox/（bwrap/seatbelt 进程隔离）
- native permissions 文件（per-harness：cursor_native_permissions/hermes_native_permissions/opencode_native_permissions/qwen_native_permissions/kiro_native_permissions/goose_native_permissions）
- server/auth.py + accounts_*（认证/账户）；AGENTS.md（仓库内 agent 行为约束）；DCO（签署）；CONTRIBUTING.md + designs/ci-external-contributors-proposal.md（治理）

## 主要外部依赖
- Modal / Daytona / Blaxel / Islo / E2B / CoreWeave / Kubernetes / OpenShell / Boxlite / microsandbox / Databricks（云沙箱 managed hosts，server/managed_hosts.py）
- 各 harness CLI（claude/codex/cursor/hermes/pi/kimi/kiro/qwen/goose/opencode/antigravity）
- SQLAlchemy（会话存储）；FastAPI（server）；bubblewrap（Linux 沙箱）；Seatbelt（macOS）

## 显著设计证据入口
- designs/CLI_CONTRACT.md、designs/harness-plugin-interface.md、designs/harness-modular-registry-proposal.md、designs/OBSERVABILITY.md、designs/CREDENTIAL_STORE.md、docs/AGENT_YAML_SPEC.md、docs/harness-bench-design.md
