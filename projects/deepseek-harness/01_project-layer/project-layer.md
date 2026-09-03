# Project Layer — deepseek-harness

> Job: `ARCH-2026-09-03-001` | 对应 `01-repository-snapshot.md`（Repository Snapshot & Project Map 的权威快照）

## 1. 项目地图（顶层）
```
apps/            cli（dsh bin）、web（Vite 前端）
docs/            architecture / capability-seams / config-catalog / subsystems(~50) / postmortem(4)
native/          landlock-run（C node-addon，Linux 沙箱 launcher）
packages/        55 组 workspace 包（core/api/llm/e2b/shell/fs/lsp/skill/web/compaction/subagent/
                bundle/interaction(approval)/sandbox/credentials/storage/session/acp/guard/...）
python/          Python SDK + bundled runtime
vendor/          vendored Cordis 框架（9 包，@deepseek-ai 作用域，link: 解析）
website/         VitePress 文档站点
scripts/         repo gates / generators（build.ts、run-gates.ts、gen-doc-graphs.ts）
snapshots/       快照测试数据
.agents/notes/   Agent Notes（implemented/architecture/ 决策记录，2026-06 起 ~40+ 条）
```
- 包规模：≈150+ workspace 包（`packages/*/*/package.json` + `vendor/*` + `apps/*` + `python/sdk-runtime`）

## 2. 生命周期（应用启动 → 会话 → 关闭）
```
dsh --profile <web|headless|sdk|sdk-minimal|acp>  →  启动强制（verify-application-entrypoints）
  → Cordis 应用树装配（bundle → profile cordis.patch.yml → home → --patch 分层覆盖）
  → session 创建（SessionHeader/EpochHeader）→ agent-loop 驱动 turn/step
  → 会话结束时 session 日志持久化（JSONL/zstd 或 SQLite）→ 关闭
```
- 会话事件流：`turn/start` → `step/start` → `user/message` → (step: `assistant/chunk` → `assistant/message` → `tool/call` → `tool/result`) → `step/end` → `turn/end`

## 3. 核心模块与职责（含 ctx 服务 key）
| 模块 | ctx key | 职责 |
|---|---|---|
| core/session | `ctx.sessions` | 内存 session 存储 + append-only SessionEvent 日志 |
| core/agent-loop | `ctx.agentLoop` | ReactLoopAgent 驱动器（turn/step 状态机） |
| core/agent | `ctx.agents` | Agent 接口/实时注册表/agent/* 事件 |
| core/tools | `ctx.tools` | 作用域工具注册 + 受控执行管线 |
| core/system-prompt | `ctx.systemPrompt` | 有序 prompt section 组装 |
| core/scope | — | per-agent 作用域注册原语 |
| llm/llm | `ctx.llm` | LLM adapter 注册表（deepseek/pi-ai/replay） |
| interaction/user-approval | `ctx.approval` | 审批 fail-closed seam |
| sandbox/sandbox | `ctx.sandbox` | 进程围栏 capability seam（bwrap/Landlock/Seatbelt/Windows ACL） |
| credentials/credentials | `ctx.credentials` | CredentialRef seam（env 变量名引用） |
| session/session-persistence-jsonl | `ctx.sessionPersistence` | JSONL 持久化（zstd） |
| storage/storage-sqlite | `ctx.storage` | 非会话存储（units/unit_globals 表，WAL） |
| subagent/* | `ctx.subagents` | 子代理多 provider（spawn/acp/claude-code/codex/dsh-sdk/fork） |
| compaction/* | `ctx.compaction` | 历史压缩为 summary 节点 |
| guard/timeout-policy | — | 工具调用超时强制器 |
| runtime-diagnostics/invariants | `ctx.invariants` | package-owned 运行时不变量注册表 |
| typert/* | `ctx.typert` | 运行时类型图（generator/loader/registry/protocol） |
| acp/* | — | Agent Client Protocol 自动化桥 |

## 4. 关键配置与约束
- `pnpm-workspace.yaml`：workspace 列表 + overrides + allowBuilds（esbuild/lefthook/node-pty/koffi）+ minimumReleaseAgeExclude + patchedDependencies
- profiles/bundles：`dsh-base` 共享首层；layer 顺序 bundle → profile → home → `--patch`
- `SESSION_FORMAT_VERSION = 0`（pinned，无迁移）；SQLite `STORAGE_SQLITE_SCHEMA_VERSION = 1`（不兼容拒绝）
- AGENTS.md 治理：pre-release stance（foundation over blast radius）、conventions（invariants 断言范围、doc-sync 门禁）

## 5. 主要外部依赖
vendored Cordis（4.0.0-rc.7）· e2b@2.29.1 · node-pty@1.2.0-beta.15 · koffi@3.1.1 · playwright@1.61.1 · @earendil-works/pi-ai · @anthropic-ai/claude-agent-sdk@0.3.241 · @openai/codex@0.149.1 · zod@^4.4.3

## 6. 测试与质量体系
- vitest 835 spec 文件；per-file 100% 覆盖率门禁；e2e（真实 API）/expected/snapshot（keyless 录制回放）/web/stress
- pytest（python/sdk）；.github/issue-management policy.test.mjs；lefthook git hooks
- CI：.github/workflows/ 20 个 + .gitlab-ci.yml

## 7. 本层事实来源
- 快照文件：`01-repository-snapshot.md`（全部字段带 `repo/` 相对路径证据）
- 本层为 Project Layer（L0），不含知识提炼；所有条目可追溯到仓库实际内容。
