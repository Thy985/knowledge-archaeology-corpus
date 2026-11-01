# 01 Project Layer — deepseek-harness v0.2.1-alpha.2（项目地图）

> 事实可回溯 `repo/`（= `archaeology-jobs/ARCH-2026-10-11-001/repo/`）；L0 项目事实。演进标注：STABLE=基线已记录且维持；CHANGED=基线有但已演进；NEW=0.2 新增。

## 1. 定位与元数据

| 项 | 值 | 证据 |
|---|---|---|
| 项目 | DeepSeek Harness（Agent Harness / Runtime） | README.md |
| 仓库 | `github.com/deepseek-ai/deepseek-harness` | AGENTS.md |
| 版本 | **0.2.1-alpha.2**（根 package.json；apps 与关键包同版本） | `repo/package.json` |
| commit / tag | `d743267`（2026-10-09 17:31:32 +0800）/ `dsh-v0.2.1-alpha.2` | `git log -1`、`git describe --tags` |
| 许可 | MIT | `repo/LICENSE` |
| 包管理 | pnpm@11.28.5 monorepo，`packages/*/*` 两层 **331 包** | `pnpm-workspace.yaml`、`find packages -name package.json -maxdepth 3` |
| 语言 | TypeScript（ESM）+ C（Landlock node-addon）+ Python SDK | package.json / python/ |
| 阶段 | developer preview，alpha 预发布（破坏性变更预期） | AGENTS.md、SAFETY.md |

## 2. 运行入口（NEW 演进：应用启动纪律）

| 入口 | 路径 | 证据 |
|---|---|---|
| dsh CLI | `apps/cli`（bin=dsh） | apps/cli/package.json |
| Web 前端 | `apps/web`（Vite） | apps/web/ |
| **桌面端（NEW）** | `apps/desktop` + `apps/desktop-host`（Electron + Desktop Host） | docs/architecture.md §Desktop |
| Python SDK | `python/sdk` + `python/sdk-runtime`（runtime wheel 打包 dsh CLI 为 deepseek-harness-sdk-runtime-<platform>-<arch>） | docs/architecture.md §Application launch |
| **启动门（NEW）** | `scripts/verify-application-entrypoints.ts`：仅 dsh profiles 可启动 Node 应用；vendored CLI/测试可执行文件/进程内挂载被显式分类拒绝 | docs/architecture.md；scripts/verify-application-entrypoints.ts |

## 3. 架构骨架（Cordis 插件树）

**五 profile**（web / headless / sdk / sdk-minimal / acp）+ `dsh-base` 共享首层 + 有序 bundle 层 + patch overlay（profile → cordis.patch.yml → home patch → --patch）。`sdk-minimal` 是唯一例外：独立显式 SDK 树，不应用 dsh-base（STABLE 架构，0.2 文档化更完整）。

核心包（ctx 键，STABLE）：

| 包 | 拥有 | ctx 键 |
|---|---|---|
| core/session | append-only SessionEvent 日志 + 内存存储 | ctx.sessions |
| core/system-prompt | prompt-section + tool-schema 组装 | ctx.systemPrompt |
| core/tools | 作用域工具注册表 + 把关执行管道 | ctx.tools |
| core/agent | Agent 接口 + 活注册表 + agent/* 事件 | ctx.agents |
| core/agent-loop | 默认驱动器 | ctx.agentLoop |
| core/scope | per-agent 作用域注册原语 | 库无键 |
| llm/llm | 消息/流词表 + 适配器 seam | ctx.llm |

证据：docs/architecture.md §Core packages。

## 4. 0.2 新增模块（演进差异）

| 模块 | 职责 | 证据 |
|---|---|---|
| ptc-runtime（ptc-runtime + ptc-runtime-node）| PTC 执行 seam：resolve/run，program 作 async body，lossless-JSON result/logs/error | packages/ptc-runtime/ptc-runtime/README.md |
| goal | 同会话持久目标服务（1 goal/session，round cap 256，CAS）| packages/goal/goal/README.md |
| schedule | host-wide 持久提醒（after/at/every/daily/weekly/cron）| packages/schedule/schedule/README.md |
| jobs | 后台任务注册表（owner=session fence，output ring，settlement notice）| packages/jobs/jobs/README.md |
| deliverables（tool-present + workspace-changes）| turn 交付记录（present 声明 + git 快照行数）| packages/deliverables/README.md |
| spill（spill-local + spill-policy）| 超大文本存储 + 预览定位器 | packages/spill/spill/README.md |
| computer-use / browser-use | 单 provider 注册（一次一个 desktop driver / browser backend）| packages/computer-use/…、browser-use/… |
| extensions（cordis-host-runner / cordis-client-runner / tool-cordis / ui-cordis）| 动态 Cordis 包（node:vm，进程级），只读 API 发现 | packages/extensions/… |
| workspace | ctx.workspaceRegistry 持久项目/会话分组（模型不可见）| packages/workspace/workspace/README.md |
| attachment / document / feedback / session-query / test-support / host / client | 附属服务与宿主支持 | packages/ 布局 |
| **sandbox 组重构（CHANGED）** | e2b 移除；sandbox / sandbox-local / sandbox-policy / sandbox-windows-acl | packages/sandbox/ |

## 5. 关键数据结构与状态

| 项 | 说明 | 证据 |
|---|---|---|
| SessionEventMap | append-only 事实源；turn/step/user/assistant/tool/request 事件；lossless JSON、seq 连续 | packages/core/session/src/types.ts |
| SessionEvent<T> 判别联合 | 插件可扩展 | 同上 |
| SessionHeader/EpochHeader | 会话/纪元头 | 同上 |
| Agent + AgentStatus | 'idle' \| 'running' | packages/core/agent/src/runtime-types.ts |
| **SESSION_FORMAT_VERSION（NEW）** | **=4**；latestFinalizedVersion 4 / latestReleasedVersion 4（evidenceTag dsh-v0.2.0-rc.2）| packages/core/session/src/types.ts:89；docs/session-format-status.md |

## 6. 测试体系

Vitest 多配置（unit/e2e/web/snapshot/expected/web-stress/web.perf/bench）；`snapshots/` 快照数据；CI 23 workflows（ci/e2e/sandbox/landlock/node-addon/python-release/pi-ai-provider-e2e/weighted-approval/issue-policy…）；`scripts/run-gates.ts` + lefthook.yml。证据：`vitest*.config.ts`、`.github/workflows/`。

## 7. 配置与治理

- `pnpm-workspace.yaml`：workspace globs + overrides（cosmokit/schemastery link vendor）
- AGENTS.md：pre-stable API 纪律、released session-format 迁移规则、application-launch 禁令
- **SAFETY.md（NEW）**：dev-preview 未审计；沙箱/审批不保证隔离；最小权限/一次性 VM/备份；"not the sole security control"
- **BENCHMARK.md（NEW）**：Python SDK jsonrpc-agent 最小变体 + 独立 workspace/session id
- 配置目录：docs/config-catalog.md（生成式权威）

## 8. 外部依赖与边界

- zod / @standard-schema/spec（schema 校验）；vendored Cordis（vendor/*，@deepseek-ai 作用域）
- native/system：Landlock launcher（C node-addon）
- 边界：sdk-minimal 例外不应用 dsh-base；schedule 不能 headless/sdk-only 单独挂载（需 Host Web Session controller + Session persistence，delivery 需 `session/flush` 确认）；archiving 含 active reminders 的会话被拒
