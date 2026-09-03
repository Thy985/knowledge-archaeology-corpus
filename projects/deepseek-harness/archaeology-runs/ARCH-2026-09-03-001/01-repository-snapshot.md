# Repository Snapshot & Project Map — deepseek-harness

> Job: `ARCH-2026-09-03-001` | 阶段：Snapshot / Project Map（**非** Knowledge Synthesis）
> 全部事实可回溯到仓库实际内容（引用路径为 `repo/` 下的相对路径）。
> 目标项目未做任何修改；本文件不写入最终 Knowledge Layer。

---

## 一、Repository Snapshot

| 字段 | 值 |
|---|---|
| **repository** | `https://github.com/deepseek-ai/deepseek-harness.git` |
| **commit SHA** | `76fda729799fe9b3848dbe2c211d4b231032b81e` |
| **branch** | `master`（浅克隆 depth=1） |
| **tag** | 无（developer preview，README/AGENTS.md 声明"第一个 tagged release 前不承诺兼容"） |
| **HEAD commit** | `Merge pull request #3481 from deepseek-harness/fix/http-proxy-rc-version`（2026-09-03 14:02:58 +0800） |
| **analysis timestamp** | 2026-09-03 20:57:11 +08（本地分析时刻） |
| **repository version** | 根 `@deepseek-ai/dsh-root` **0.1.2-rc.1**（`repo/package.json`）；CLI `@deepseek-ai/dsh` 0.1.2-rc.1（`repo/apps/cli/package.json`）；Web `@deepseek-ai/dsh-web-frontend` 0.1.2-rc.1；native `@deepseek-ai/node-addon-landlock-run-workspace` 0.1.1 |
| **knowledge-archaeology-skill version** | **v3.2**（本地已装，与 skill 仓库 SKILL.md 逐字节一致；skill repo commit `9399731`） |
| **工程元数据** | pnpm monorepo，`packageManager: pnpm@11.7.0`，`engines.node: ^22.19.0 || >=24.0.0`，`type: module`，LICENSE MIT（`repo/package.json`, `repo/LICENSE`） |

---

## 二、Repository 目录地图

```
deepseek-harness/
├── AGENTS.md / CLAUDE.md(→AGENTS.md)  # 仓库治理：pre-release stance、仓库布局、命令
├── .agents/                           # Agent 工作流 + Agent Notes（notes/ 决策记录）
├── .claude/skills/                    # Claude 技能
├── apps/                              # 产品组装层
│   ├── cli/                           #   dsh CLI（bin=dsh → lib/bin.js，src/bin.ts）
│   └── web/                           #   Web 前端（Vite，index.html）
├── docs/                              # 架构文档 + 子系统文档（i18n 中英）
│   ├── architecture.md / capability-seams.md / cordis-primer.md
│   ├── tool-execution-pipeline.md / event-producer-consumer.md / graph-atlas.md
│   ├── module-graph.md / persistence-catalog.md / config-catalog.md / tool-catalog.md
│   ├── agent-lifecycle.md / defensive-patterns.md / testing.md / development.md
│   └── subsystems/                    #   ~50 个子系统说明（approval/sandbox/credentials/...）
├── native/landlock-run/               # Linux 沙箱 launcher（C node-addon，entry/src/main.c）
├── packages/<group>/<pkg>/            # 55 组 workspace 包（每组含多包，≈150+ 包）
│   ├── core/                          #   产品 API 脊柱（agent/agent-loop/session/...）
│   ├── api/  typert/  llm/  e2b/  shell/  subprocess/  terminal/  fs/  lsp/  skill/
│   ├── web/  compaction/  context/  subagent/  bundle/  workflow/  webhook/  todo/
│   ├── plan/  preset/  guard/  hooks/  session/  identity/  settings/  credentials/
│   ├── acp/  interaction/  boot/  sdk/  storage/  sandbox/  experimental/  util/  ...
├── python/                            # Python SDK + bundled runtime（uv + pyproject）
├── scripts/                           # repo gates / generators（build.ts、run-gates.ts 等）
├── snapshots/                         # 快照测试数据
├── vendor/                            # vendored Cordis 框架源（9 包，@deepseek-ai 作用域）
├── website/                           # VitePress 文档投影
├── patches/                           # pnpm patchedDependencies（pkg/node-pty）
├── package.json / pnpm-workspace.yaml / pnpm-lock.yaml
├── tsconfig*.json / vitest*.config.ts / .oxlintrc*.json / .jscpd.json / lefthook.yml
├── .github/workflows/                 # 20 个 CI workflow（ci/e2e/sandbox/landlock/python-release...）
├── .gitlab-ci.yml / pytest.ini / SAFETY.md / THIRD_PARTY_NOTICES.md
```

---

## 三、主要语言

| 语言 | 用途 | 证据 |
|---|---|---|
| **TypeScript**（主） | 全部 packages/ 与 apps/，ESM，`type: module` | `repo/package.json` |
| **C** | native landlock-run node-addon（Linux 沙箱入口） | `repo/native/landlock-run/packages/entry/src/main.c` |
| **Python** | Python SDK + runtime wheel | `repo/python/sdk/pyproject.toml`, `uv.lock` |
| **YAML** | Cordis 配置（cordis.yml / patch overlay） | `repo/apps/cli/src/sdk-source.cordis.patch.yml` 等 |
| HTML/TS（Web） | Vite 前端 | `repo/apps/web/index.html` |

---

## 四、主要运行入口

| 入口 | 说明 | 证据 |
|---|---|---|
| **`dsh` CLI** | 唯一受支持的 Node 应用启动器；`#!/usr/bin/env node`；bin 名 `dsh` → `lib/bin.js` | `repo/apps/cli/src/bin.ts`, `repo/apps/cli/package.json` |
| **5 个 profile** | `web`（Web UI，默认 127.0.0.1:3080）、`headless`（一次性、无 server）、`sdk`、`sdk-minimal`、`acp`（自动化 ACP server） | `repo/docs/architecture.md` §Profiles；`repo/README.md` |
| **启动命令** | `npx @deepseek-ai/dsh web` / `dsh --profile <name>` / `dsh --profile web --dump-config`（查看插件树） | `repo/README.md` |
| **Python SDK** | 客户端默认启动 `dsh --profile sdk`，runtime wheel 打包 `dsh` CLI | `repo/docs/architecture.md` §Application launch |
| **应用启动强制规则** | `scripts/verify-application-entrypoints.ts` 拒绝绕过 `dsh` 的 Node 应用路径 | `repo/docs/architecture.md`；AGENTS.md |

---

## 五、核心模块（Cordis 插件树）

| 包（ctx key） | 职责 | 证据 |
|---|---|---|
| `core/session`（`ctx.sessions`） | append-only `SessionEvent` 日志 + 内存存储；`deriveMessages()` 投影模型历史 | `repo/docs/architecture.md` §Core packages；`repo/packages/core/session/src/index.ts` |
| `core/system-prompt`（`ctx.systemPrompt`） | prompt section + tool schema 组装 | 同上 |
| `core/tools`（`ctx.tools`） | 作用域工具注册表 + guarded 执行管线（pre/post-execute） | 同上；`docs/tool-execution-pipeline.md` |
| `core/agent`（`ctx.agents`） | `Agent` 接口、实时注册表、`agent/*` 事件 | 同上 |
| `core/agent-loop`（`ctx.agentLoop`） | 默认驱动器（`agent.ts`：phase 状态机 + tool 执行） | `repo/packages/core/agent-loop/src/agent.ts` |
| `core/scope` | per-agent 作用域注册原语（库，无 ctx key） | `repo/docs/architecture.md` |
| `llm/llm`（`ctx.llm`） | 消息/流词汇 + adapter seam（DeepSeek 等 provider） | 同上 |
| `webhook/webhook`（`ctx.webhookRuntime`） | 认证投递 dispatch + Workspace Session 创建 | 同上 |
| 能力组 | shell/bash、subprocess、terminal、fs、lsp、skill、web、e2b、compaction、subagent、workflow、todo、plan、preset、guard、hooks、session(持久化/projection/telemetry)、identity、settings、credentials、acp、interaction、boot、sdk、storage(sqlite)、sandbox、api/typert | `repo/AGENTS.md` §Repository layout |

---

## 六、核心数据结构

| 结构 | 要点 | 证据 |
|---|---|---|
| **`SessionEventMap`** | merge-extensible、append-only 事实源；成员如 `turn/start`、`turn/end`、`step/start`、`step/end`、`user/message`、`assistant/chunk`、`assistant/message`、`tool/call`、`tool/result`、`request/header`；lossless JSON、seq 连续 | `repo/packages/core/session/src/types.ts:261` 起 |
| **`SessionEvent<T>`** | 判别联合；`SessionEventType = keyof SessionEventMap`；插件可扩展 | `repo/packages/core/session/src/types.ts:368,436` |
| **`SessionHeader` / `EpochHeader`** | 会话/纪元头 | `repo/packages/core/session/src/types.ts:92,224` |
| **`Agent` 接口 + `AgentStatus`** | `'idle' \| 'running'` | `repo/packages/core/agent/src/types.ts`, `runtime-types.ts:53` |
| **`ApprovalRequestId`**（branded） | 区分审批 id 与 tool-call/agent/session id | `repo/docs/subsystems/approval.md` |
| **`ApprovalOutcome`** | 闭合 fail-closed：`'allowed-once'\|'rejected'\|'cancelled'\|'unavailable'` | 同上 |
| **`ApprovalPolicy`** | `'ask' \| 'never'`（never = 确定性拒绝，严格 headless） | 同上 |
| **`SandboxMode`** | `'read-only'\|'workspace-write'\|'danger-full-access'`（只管文件效应） | `repo/docs/subsystems/sandbox.md` |
| **`CredentialRef`**（branded）+ `ResolvedCredential` | 环境变量名引用；`{value, source}`，每操作重解析 | `repo/docs/subsystems/credentials.md` |
| **`PresetSpec`** | sandbox+approval 组合（预设表） | `repo/docs/subsystems/permission-presets.md` |

---

## 七、核心状态

| 状态 | 机制 | 证据 |
|---|---|---|
| **session log** | append-only 不可变事实源；**"model-visible means logged"** 运行时不变量；replay/持久化/telemetry 全由此派生 | `repo/docs/architecture.md` §Session log；`repo/packages/core/session/src/invariant.ts` |
| **turn/step 生命周期** | step = 一次 model request + 其工具调用；turn = 0+ steps；`turn/*`、`step/*` 为持久事件 | `repo/docs/architecture.md` §Turn flow |
| **session projection** | `ctx.sessionProjections`：注册单元增量折叠已提交事件，`stateOf()`/`snapshot()`；agent-loop 注册 `turnBoundary` 共享状态 | `repo/docs/architecture.md`；`docs/subsystems/session-projection.md` |
| **agent 状态** | `idle/running`；`agent/*` 事件：inbox、step、status、request、validation、continuation | `repo/packages/core/agent/src/runtime-types.ts` |
| **per-session 策略** | `approval/policy` 与 `sandbox/mode` 以 session log 事件持久化，重放可重建；`setApprovalPolicy()` 单一写路径 | `repo/docs/subsystems/approval.md`, `sandbox.md` |
| **版本状态** | `SESSION_FORMAT_VERSION` pinned at `0`（无迁移承诺）；SQLite `SCHEMA_VERSION`（monotonic） | `repo/packages/core/session/README.md:181`；`repo/packages/storage/storage-sqlite/src/schema.ts` |

---

## 八、主要测试体系

| 体系 | 说明 | 证据 |
|---|---|---|
| **vitest（unit）** | 835 个 spec/test 文件；`pnpm test`；coverage gate **per-file 100%**（`test:coverage`，CI 强制） | `repo/package.json` scripts；实测 `find packages apps -name '*.spec.ts' -o -name '*.test.ts' \| wc -l` = 835 |
| **e2e** | `test:e2e` 真实 API，无 `DEEPSEEK_API_KEY` 自跳过；另有 `e2e.yml` CI | `repo/package.json` |
| **expected / snapshot** | `test:expected`（owner-local 过程期望）、`test:snapshot`（keyless 录制回放，`DSH_SNAPSHOT=record/refresh`） | `repo/package.json`；`vitest.snapshot.config.ts` |
| **web 测试** | `test:web` / `stress-tests`（Vite） | `repo/apps/web/stress-tests/`, `vitest.web*.config.ts` |
| **pytest（Python SDK）** | `python/sdk/tests`，`testpaths` 限定 | `repo/pytest.ini` |
| **issue-management** | `.github/issue-management/policy.test.mjs`（issue 策略门禁） | `repo/package.json`；`.github/workflows/issue-policy.yml` |
| **CI** | `.github/workflows/` 20 个（ci/sandbox/landlock-run/python-release/release 等）+ `.gitlab-ci.yml` | `repo/.github/workflows/` |
| **git hooks** | lefthook.yml | `repo/lefthook.yml` |
| **静态/质量** | oxlint（.oxlintrc.json）、jscpd 克隆检测（.jscpd.json）、hygiene/publint、doc-sync 门禁 | `repo/package.json` scripts |

---

## 九、主要配置

| 配置 | 内容 | 证据 |
|---|---|---|
| **pnpm-workspace.yaml** | workspace 列表（vendor/*, packages/*/*, apps/*, website, native, python/sdk-runtime）；overrides；`allowBuilds`（esbuild/lefthook/node-pty/koffi）；`minimumReleaseAgeExclude`；`patchedDependencies` | `repo/pnpm-workspace.yaml` |
| **profiles + bundles** | `web/headless/sdk/sdk-minimal/acp`；bundle = Cordis config rows + code；`dsh-base` 为共享首层；layer 顺序：bundle 列表 → profile `cordis.patch.yml` → home 级 → `--patch` | `repo/docs/architecture.md` §Profiles and bundles |
| **tsconfig 体系** | base/host/client/web（`tsconfig*.json`） | `repo/` 根 |
| **lint** | `.oxlintrc.json` / `.oxlintrc.staged.json` | `repo/` 根 |
| **settings / credentials** | 用户设置能力；credential 经 env/.env provider | `repo/docs/subsystems/settings.md`, `credentials.md` |
| **config catalog** | 生成的全量配置目录 | `repo/docs/config-catalog.md` |
| **SAFETY** | 实验性声明：未安全审计、沙箱非唯一安全控制 | `repo/SAFETY.md` |

---

## 十、主要权限 / Policy / Governance 机制

| 机制 | 要点 | 证据 |
|---|---|---|
| **Approval seam** | `ctx.approval` 单一 dispatch；`approval/request` 回答者瀑布；fail-closed（非 `allowed-once` 即拒绝）；`approval/asked`+`approval/decided` 审计事件对 | `repo/docs/subsystems/approval.md` |
| **ApprovalPolicy** | per-session `ask`/`never`；`never` = 无提示确定性拒绝（CI/unattended） | 同上 |
| **Sandbox** | Linux bwrap/Landlock（native C launcher）、macOS Seatbelt、Windows ACL restricted-token；`read-only`/`workspace-write`/`danger-full-access`；enforcement `full/partial` 作为上报事实 | `repo/docs/subsystems/sandbox.md`；`repo/native/landlock-run/` |
| **Permission Presets** | `ctx.permissionPresets` 把 sandbox+approval 两个旋钮打包为命名预设（默认 `workspace-write`+`ask`、`danger-full-access`+`never`）；自身不执行强制 | `repo/docs/subsystems/permission-presets.md` |
| **Credentials** | 秘密不进配置；引用即环境变量名；每操作重解析（轮换热生效）；空值处处视为不存在 | `repo/docs/subsystems/credentials.md` |
| **Guard** | loop-hygiene + tool-timeout 插件（loop 卫生/工具超时） | `repo/AGENTS.md` layout |
| **ACP automation bridge** | 自动化 ACP server 提供一次性机器决策（`acp` profile） | `repo/docs/subsystems/approval.md` |
| **启动强制** | `verify-application-entrypoints.ts`：所有 bin/demo 显式分类，拒绝绕过 `dsh` 的应用路径 | `repo/docs/architecture.md`；AGENTS.md |
| **仓库治理** | AGENTS.md（pre-release stance: foundation over blast radius）、CONTRIBUTING、BRAND_GUIDELINES、`.github/issue-management` + issue-policy、docs/postmortem | `repo/` 根 |
| **安全立场** | SAFETY.md：未审计、实验性、不保证隔离；建议最低权限/一次性环境 | `repo/SAFETY.md` |

---

## 十一、主要外部依赖

| 依赖 | 用途 | 证据 |
|---|---|---|
| **vendored Cordis 框架**（9 包） | `@deepseek-ai/cordis` 4.0.0-rc.7 + loader/include/group/timer/hmr/logger-console + cosmokit 1.8.1 + schemastery 3.18.0；源 vendored、`link:` 解析 | `repo/vendor/README.md`（manifest 表） |
| **e2b@2.29.1** | E2B 沙箱集成（remote sandbox 能力） | `repo/pnpm-lock.yaml` |
| **node-pty@1.2.0-beta.15** | 持久 PTY 后端（Windows ConPTY） | `repo/pnpm-lock.yaml`；pnpm-workspace allowBuilds |
| **koffi@3.1.1** | Windows FFI（JSONL 写透传 MoveFileExW） | `repo/pnpm-workspace.yaml`（allowBuilds）；lock |
| **playwright@1.61.1** | 浏览器自动化（web capability） | `repo/pnpm-lock.yaml` |
| **@earendil-works/pi-ai** | 可选 LLM API 后端（模型目录更新） | `repo/pnpm-workspace.yaml`（minimumReleaseAgeExclude） |
| **@anthropic-ai/claude-agent-sdk@0.3.241** | subagent 驱动（外部 agent 委托） | `repo/pnpm-workspace.yaml`（release-age 豁免） |
| **@openai/codex@0.149.1** | subagent 驱动（外部 agent 委托） | 同上 |
| **zod@^4.4.3 / @standard-schema/spec** | schema 校验 | `repo/packages/core/agent-loop/package.json`；vendor README |
| **工具链** | typescript、tsx、tsdown、vitest、oxlint、jscpd、Vite（web） | `repo/package.json` |

---

## 附：事实来源与边界

- 所有条目均已对照 `repo/`（本地只读克隆，HEAD `76fda72`）实际内容核实；文档类声明与源码实现的关系属后续 Discovery/Synthesis 阶段的核验范围，本快照仅记录"仓库呈现的事实"。
- 未修改目标项目、未写入 Corpus、未进入 Knowledge Layer。
