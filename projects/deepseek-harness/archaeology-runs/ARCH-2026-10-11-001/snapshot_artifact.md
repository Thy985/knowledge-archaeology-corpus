# Repository Snapshot & Project Map — deepseek-harness（v0.2.1-alpha.2）

> Job: `ARCH-2026-10-11-001` | 阶段：Snapshot / Project Map（**refresh**，基线 = ARCH-2026-09-03-001 @ 0.1.2-rc.1/`76fda729`）
> 全部事实可回溯到仓库实际内容（引用路径为 `repo/` 相对路径，即 `archaeology-jobs/ARCH-2026-10-11-001/repo/`）。

## 一、Repository Snapshot

| 字段 | 值 |
|---|---|
| **repository** | `https://github.com/deepseek-ai/deepseek-harness.git` |
| **commit SHA** | `d743267388641bc76f17c45ce8b4c231aed1d32c` |
| **branch** | `master`（浅克隆 depth=1） |
| **tag** | **`dsh-v0.2.1-alpha.2`**（`git describe --tags` 直接命中——对比基线 0.1.2-rc.1 无 tag 命中，0.2 阶段开始打 tag） |
| **HEAD commit** | `Merge pull request #5946 from deepseek-harness/release-0.2.1-alpha.2`（2026-10-09 17:31:32 +0800，author 07akioni） |
| **analysis timestamp** | 2026-10-10T18:14:09Z（UTC 实测） |
| **repository version** | 根 `@deepseek-ai/dsh-root` **0.2.1-alpha.2**（`repo/package.json`）；apps 全部 0.2.1-alpha.2（cli/web/desktop/desktop-host）；关键包 ptc-runtime/computer-use/browser-use/goal/schedule/spill/workspace/jobs/attachment 均为 0.2.1-alpha.2 |
| **skill version** | knowledge-archaeology **v3.2**（生产版 `/home/user/.super_doubao/super-doubao-runtime/workspace/.user_skills/knowledge-archaeology/`） |
| **工程元数据** | pnpm monorepo，`packageManager: pnpm@11.28.5`（基线 11.7.0），`type: module`，LICENSE MIT（`repo/package.json`, `repo/LICENSE`） |

## 二、仓库规模与结构演进（vs 0.1.2-rc.1）

| 维度 | 0.1.2-rc.1（2026-09-03） | 0.2.1-alpha.2（2026-10-09） | 证据 |
|---|---|---|---|
| workspace 包数 | ≈150+（55 组，`packages/*/*`） | **331 个** `package.json`（`packages/*/*` 两层） | `find packages -name package.json -maxdepth 3 \| wc -l` |
| 顶层结构 | AGENTS.md / apps(cli,web) / docs / native / packages / python / scripts / snapshots / vendor / website / patches | 同 + `benchmarks/`、`native/system`（Landlock launcher 移入 native/system） | `ls` |
| apps | cli + web | **cli + web + desktop + desktop-host**（桌面端新增） | `apps/` |
| 文档 | docs/architecture.md 等 + subsystems ~50 | 同 + **SAFETY.md / BENCHMARK.md / docs/session-format-status.md**（BENCHMARK.md 新增） | `ls docs` 根、`SAFETY.md` |
| CI | 20 workflows | **23 workflows**（新增 pi-ai-provider-e2e / weighted-approval* / issue-lifecycle / issue-policy 等治理类） | `.github/workflows/` |

## 三、packages/ 演进差异（vs 0.1.2-rc.1 基线列表）

**新增包组/包**（基线未列）：`attachment`、`browser-use`、`client`、`computer-use`、`deliverables`、`document`、`extensions`（运行时自修改）、`feedback`、`goal`（会话目标）、`host`、`jobs`（后台任务，基线有 jobs? 旧地图未列）、`mcp`（基线内置 mcp 但未独立列出，现为独立组）、`ptc-runtime`（ptc-runtime + ptc-runtime-node，PTC 执行独立运行时）、`schedule`（定时跟进）、`session-query`、`spill`、`test-support`、`workspace`（工作区）。

**移除**：`e2b`（基线 e2b 沙箱包组消失——`find packages -maxdepth 2 -type d -name "*e2b*"` 无结果；沙箱职责由 `sandbox/` 组承担：sandbox / sandbox-local / sandbox-policy / sandbox-windows-acl）。

**保留（核心面未变）**：core（agent/session API）、api、typert、llm、shell、subprocess、ssh、terminal、fs、lsp、skill、web、compaction、context、subagent、bundle、workflow、todo、plan、preset、guard、session、identity、settings、credentials、acp、interaction、boot、sdk、storage、sandbox、experimental、util、telemetry。

## 四、主要语言

| 语言 | 说明 | 证据 |
|---|---|---|
| **TypeScript**（主） | 全部 packages/ 与 apps/，ESM `type: module` | `repo/package.json` |
| **C** | Landlock launcher node-addon（迁至 `native/system`） | `repo/native/system/packages/entry/src/main.c`（基线 `native/landlock-run/`） |
| **Python** | Python SDK + bundled runtime（uv + pyproject；`python/sdk` + `python/sdk-runtime`） | `repo/python/` |

## 五、主要运行入口

| 入口 | 路径 | 证据 |
|---|---|---|
| dsh CLI | `apps/cli`（bin=dsh） | `repo/apps/cli/package.json` |
| Web 前端 | `apps/web`（Vite） | `repo/apps/web/` |
| 桌面端 | `apps/desktop` + `apps/desktop-host`（新增） | `repo/apps/desktop/` |
| 根构建/门 | `scripts/run-gates.ts`、`Makefile` | `repo/scripts/`、`repo/Makefile` |

## 六、核心模块（Cordis 插件树，架构承诺维持）

| 模块 | 职责 | 证据 |
|---|---|---|
| `core/session`（`ctx.sessions`） | append-only `SessionEvent` 日志；`deriveMessages()` 投影 | `repo/packages/core/session/src/index.ts`；`docs/architecture.md` |
| `core/agent-loop`（`ctx.agentLoop`） | 默认驱动器（phase 状态机 + tool 执行） | `repo/packages/core/agent-loop/src/agent.ts` |
| `ptc-runtime`（**新增**） | PTC 执行独立运行时（ptc-runtime / ptc-runtime-node） | `repo/packages/ptc-runtime/ptc-runtime/package.json` |
| `sandbox/`（e2b 移除后主体） | 进程收容：sandbox / sandbox-local / sandbox-policy / sandbox-windows-acl | `repo/packages/sandbox/` |
| `guard` | loop/tool 守卫 | `repo/packages/guard/` |
| `extensions`（**新增**） | 运行时自修改 | `repo/packages/extensions/` |
| `goal` / `schedule` / `jobs`（**新增**） | 会话目标 / 定时跟进 / 后台任务 | `repo/packages/{goal,schedule,jobs}/` |
| `computer-use` / `browser-use`（**新增**） | 计算机交互 / 浏览器交互 | `repo/packages/{computer-use,browser-use}/` |
| `mcp`（独立组） | MCP 客户端 | `repo/packages/mcp/` |
| `acp` | Agent Client Protocol | `repo/packages/acp/` |

## 七、核心数据结构（维持基线）

| 结构 | 说明 | 证据 |
|---|---|---|
| `SessionEventMap` | merge-extensible、append-only 事实源；turn/step/user/assistant/tool/request 事件；lossless JSON、seq 连续 | `repo/packages/core/session/src/types.ts` |
| `SessionEvent<T>` | 判别联合；插件可扩展 | `repo/packages/core/session/src/types.ts` |
| `SessionHeader` / `EpochHeader` | 会话/纪元头 | 同上 |
| `Agent` + `AgentStatus` | `'idle' \| 'running'` | `repo/packages/core/agent/src/runtime-types.ts` |

## 八、核心状态与不变量

| 状态 | 说明 | 证据 |
|---|---|---|
| session log | append-only 不可变事实源；**"model-visible means logged"** | `docs/architecture.md` §Session log；`packages/core/session/src/invariant.ts` |
| agent 状态 | idle/running；agent/* 事件（inbox/step/status/request/validation/continuation） | `packages/core/agent/src/runtime-types.ts` |
| 版本状态 | **`docs/session-format-status.md` 显式化**（基线 README 内嵌）：released session format 迁移用 adjacent version-named successor，禁止覆盖已提交世代 | `docs/session-format-status.md`；`AGENTS.md` |
| 版本化 schema | SQLite 单调 `SCHEMA_VERSION` | `packages/storage/storage-sqlite/src/schema.ts` |

## 九、测试体系（强化）

| 体系 | 说明 | 证据 |
|---|---|---|
| Vitest 多配置 | vitest.config / vitest.e2e / vitest.web / vitest.snapshot / vitest.expected / vitest.web-stress / vitest.web.perf / vitest.bench | `repo/vitest*.config.ts` |
| 快照测试 | `snapshots/` 数据 + vitest.snapshot.config | `repo/snapshots/` |
| E2E | `.github/workflows/e2e.yml` + pi-ai-provider-e2e.yml（新增） | `.github/workflows/` |
| 沙箱 CI | sandbox.yml / node-addon-system.yml | `.github/workflows/` |

## 十、主要配置

| 配置 | 说明 | 证据 |
|---|---|---|
| pnpm-workspace.yaml | workspace globs（vendor/*, packages/*/*, native/system, apps/*, benchmarks, website, python/sdk-runtime）；overrides（cosmokit/schemastery link vendor） | `repo/pnpm-workspace.yaml` |
| tsconfig 家族 | base/client/host/root + tsdown.config.ts | `repo/tsconfig*.json` |
| lefthook.yml | git hooks | `repo/lefthook.yml` |

## 十一、权限 / Policy / Governance 机制（0.2 显式化）

| 机制 | 说明 | 证据 |
|---|---|---|
| **SAFETY.md**（新增） | 安全策略文档（0.2 阶段显式化） | `repo/SAFETY.md` |
| `ctx.sandbox` / sandbox-policy / approval / permission-presets / credentials | 全套 seam（维持基线） | `packages/sandbox/`、`packages/credentials/` |
| AGENTS.md 治理 | pre-stable API 纪律、released session format 迁移规则、application-launch 禁令（仅 dsh profiles 启动应用） | `repo/AGENTS.md` |
| CI 治理 workflow | weighted-approval / issue-policy / issue-lifecycle（新增治理类） | `.github/workflows/` |

## 附：refresh 关注面（对照 10-11 雷达重大事件日 11）

- **评测隔离**：dsh 作为 eval harness，其网络策略/沙箱配置是否 fail-closed、评测期间外部访问的护栏——10-11 雷达明确"DeepSeek Harness 验证计划需加入隔离性检查"
- **Claude Code Mods 兼容层**：v0.2.1-alpha.1 声明"Mods API 是 dsh 插件体系子集"——需验证 `extensions`/`skill` 包中的实现与声明一致性
- **"无特权内核"承诺**：0.2 阶段是否维持（新增 extensions 自修改包是否引入特权面）

## 事实边界

- 版本号/tag/commit 来自 `git rev-parse`/`git describe`/`package.json`（实测）
- 目录/包列表来自 `ls`/`find`（实测）
- 不变量描述来自 docs/architecture.md 与源码（正文引用路径可回溯）
- 与基线的差异标注以 corpus `projects/deepseek-harness/archaeology-runs/ARCH-2026-09-03-001/01-repository-snapshot.md` 的包列表为对照
