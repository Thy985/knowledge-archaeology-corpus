# Repository Snapshot — pullfrog (ARCH-2026-10-02-001)

## 1. 仓库元数据
| 字段 | 值 |
|---|---|
| repository | https://github.com/pullfrog/pullfrog.git |
| HEAD commit | `d287f81e93e245845aa25b0916417a4216f37dad` |
| branch | main（默认分支） |
| commit message | "no-key error: name a subscription the runner could not install (#1452)" |
| commit date | 2026-10-01 18:10:14 UTC |
| version | 0.1.94（package.json） |
| cliContractVersion | 1 |
| license | MIT（Copyright 2026 Pullfrog, Inc.） |
| 快照时间 | 2026-10-02 02:12 UTC |
| 克隆方式 | git clone --depth 1（浅克隆，1 commit） |
| 文件数 | 279 个 tracked files |

## 2. 语言构成
| 类型 | 数量 |
|---|---|
| TypeScript (.ts) | 243 |
| YAML (.yml/.yaml) | 12 |
| JSON | 6 |
| Markdown | 4 |
| 其他（snap/txt/sh/grit/npmrc） | 14 |

## 3. 项目定位（README 事实）
- **一句话定位**：`The BYOK CodeRabbit that runs in your GitHub Actions`——BYOK（Bring Your Own Key/Subscription）AI 代码审查机器人，运行在 GitHub Actions 中
- **关键事实**：**Pullfrog 不是 agent 本身**——它包装 vanilla Claude Code / Codex / OpenCode，按配置选择匹配的 vendor agent 运行
- 能力：审查新 PR / 响应 review 评论 / 修复 CI 失败 / 解决 merge 冲突 / issue triage / ad-hoc（@pullfrog 标签触发）
- 运行载体：`pullfrog.yml` workflow 使用开源 action；或用 `pullfrog/pullfrog@v0` 作独立步骤（headless）
- 模型接入：订阅（Claude Pro/Max、ChatGPT Codex、Grok、Kimi Code、OpenCode Go）或 BYO API key / 内置 router（raw cost 无加价）
- 免费策略：个人账号与开源 repo 免费；Pro $30/月/org（私有 repo）
- 开源仓库 URL：pullfrog/pullfrog，★1,269（2026-10-02 API 实测），MIT

## 4. 入口
| 入口 | 文件 | 说明 |
|---|---|---|
| GitHub Action main | `entry.ts` → `main.ts` (1102 行) | action 主流程；post 步骤 `entryPost.ts`（always() 运行，持久化 auth 刷新等状态） |
| GitHub Action post | `entryPost.ts` (30 行) | Codex auth.json refresh write-back |
| CLI | `cli.ts` → `runCli.ts` (362 行) | `pullfrog` / `pf` / `pullfrog-dev` bin |
| Docker 测试 | `docker.ts` (532 行) | GHA 模拟容器（ubuntu:24.04 + node24 + GHA 工具集） |
| action.yml inputs | prompt / prompt_file / timeout / model / effort / debug / cwd / push / shell / status_checks / progress_comments / output_schema / token | |

## 5. 核心模块地图
| 模块 | 行数 | 职责 |
|---|---|---|
| models.ts | 2105 | 模型目录、选择、解析（curated slug + raw models.dev specifier）、跨 vendor 映射 |
| main.ts | 1102 | action 主逻辑：事件→agent 运行编排 |
| docker.ts | 532 | GHA 测试容器构建/运行（content-hash 门控重建） |
| external.ts | 527 | 外部 agent 进程（Claude Code/Codex/OpenCode）生命周期管理 |
| modes.ts | 441 | 运行模式定义（review/triage/fix 等） |
| configuration.ts | 421 | 仓库级配置（repo settings / per-trigger 指令） |
| toolState.ts | 356 | 工具状态跟踪 |
| runCli.ts | 362 | CLI 运行编排 |
| effort.ts | 140 | 推理 effort 映射（low..max ↔ [0,1]） |
| agents/ | — | claude.ts / codex.ts / opencode.ts / reviewer.ts / gateServer.ts / claudePretoolGate.ts / nativeFsDenies.ts / opencodePlugin.ts |
| mcp/ | — | GitHub MCP server：git.ts / gh.ts / comment.ts / checkout.ts / issueEvents.ts / issueComments.ts / checkSuite.ts / commitInfo.ts / currentObject.ts / dependencies.ts / geminiSanitizer.ts |
| commands/ | — | auth / config / console / gha / init / mcp / secret / watch（CLI 子命令） |
| utils/ | 105 文件 | agent.ts / run.ts / shell.ts / secrets.ts / token.ts / apiKeys.ts / codexAuth.ts / gitAuth.ts / gitAuthServer.ts / learnings.ts / worktree.ts / credentialsPool.ts / subscriptionProbe.ts / providerErrors.ts / secretCommands.ts / proxy.ts / browser.ts / autoMerge.ts / rangeDiff.ts / payload.ts / runContext.ts / statusChecks.ts 等 |
| prep/ | — | installNodeDependencies.ts / installPythonDependencies.ts（agent 前置依赖安装） |
| skills/git-archaeology | — | 内置 skill |
| get-installation-token/ | — | 独立 action：短期 installation token 铸造（action.yml + entry.ts + post.ts） |

## 6. 核心数据结构（从代码导出）
- **Prompt payload**：string 或 JSON payload（action input）
- **RunContext**（runContext.ts / runContextData.ts）：运行上下文（repo/PR/trigger 类型）
- **ToolState**（toolState.ts）：agent 工具调用状态跟踪
- **CredentialPool / subscriptionCredentials**（utils）：订阅凭据池
- **Models catalog**（models.ts）：vendor→model→provider 映射 + curated slugs
- **Modes**（modes.ts）：自动化模式定义（PR review / issue triage / CI fix / merge conflict / ad-hoc）

## 7. 状态 / 生命周期
- **Action 生命周期**：main step（entry.ts）→ post step（entryPost.ts，`post-if: always()`，抗 cancellation/timeout/unhandled error）
- **运行生命周期**：runLifecycle.ts / run.ts / exitHandler.ts / lifecycle.ts——启动日志→agent 运行→结果 post→stop hook
- **凭证生命周期**：installation token 短时铸造（get-installation-token action）→ 运行结束自动吊销（README 声明）
- **Hooks**：setup / post-checkout / pre-push / stop（在 agent 权限边界内运行；stop 非零退出→以失败上下文恢复 agent 自修）

## 8. 测试体系
| 类别 | 位置 | 内容 |
|---|---|---|
| 单元测试 | 顶层 *.test.ts + 各模块 *.test.ts | vitest；models.test.ts / effort.test.ts / entryPost.stdlibOnly.test.ts |
| 集成测试 | test/ci.test.ts、test/matrix.ts、test/run.ts | CI 矩阵 |
| 场景分类测试 | test/agnostic/ | gitHooks / gitPerms / missingKeyError / packageJsonScripts / pushDisabled / pushEnabled / pushRestricted / timeout |
| 跨 agent 测试 | test/crossagent/ | bedrockClaude / codexAuth / fsExfil / gitNativeWrite / mcpmerge |
| 对抗测试 | test/adhoc/ | askpassIntercept / gitExecBypass / gitFlagInjection / nobashcreative / pushRestrictedAdversarial / requirementsTxtAttack |
| 模型目录测试 | test/models-catalog.main.test.ts、test/model-smoke.ts、test/smoke-models.ts | 模型目录冒烟 |
| 容器化测试 | docker.ts + Dockerfile | GHA 等同环境（ubuntu:24.04 + gh/jq/git/python3/ssh） |
| CI workflows | .github/workflows/{test,publish,pullfrog,test-token,trigger-sync}.yml | |

## 9. 配置
- **action.yml inputs**：prompt / prompt_file / timeout（默认 1h）/ model / effort / debug / cwd / push（disabled|restricted|enabled，默认 enabled）/ shell（disabled|restricted|enabled；公共 repo 默认 restricted，私有默认 enabled）/ status_checks（deprecated→console）/ progress_comments / output_schema（JSON Schema draft-07 结构化输出）/ token
- **仓库级配置**：configuration.ts（repo settings，console 可编辑）
- **.github/workflows/pullfrog.yml**：官方模板（workflow_dispatch + id-token:write + contents:read）
- **models.ts**：curated model slugs + raw models.dev specifier

## 10. 权限与治理机制
- **最小权限 workflow 模板**：`id-token: write` + `contents: read`（官方 pullfrog.yml）
- **短期 token**：get-installation-token action 铸造安装 token，运行完成自动吊销
- **Shell 隔离**：shell.ts——命令在隔离子进程运行，restricted 模式过滤敏感环境变量（secrets.ts / secretCommands.ts / normalizeEnv.ts）
- **Key masking**：日志自动脱敏（README + providerErrors.ts）
- **Push 分级**：disabled（只读）/ restricted（仅 feature branch，禁默认分支/删分支/tag push）/ enabled
- **权限边界 hooks**：setup/post-checkout/pre-push/stop 在 agent 权限边界内
- **对抗测试**：test/adhoc/* 验证权限边界不被绕过（askpass 拦截、git exec 绕过、git flag 注入、push restricted 对抗、requirements.txt 攻击）

## 11. 外部依赖
- @anthropic-ai/claude-code 2.1.284 / @openai/codex 0.159.2 / opencode-ai 1.18.29（包装的 agent）
- @modelcontextprotocol/sdk 1.26.0 / fastmcp 3.34.0（MCP server）
- @octokit/rest 22.0.0 + webhooks-types 7.6.1（GitHub API）
- agent-browser 0.25.4（headless browser：E2E 测试/截图/UI 迭代）
- arktype 2.2.0 / zod 4.3.6（schema 校验）/ yaml / turndown / esbuild / vitest 4.0.17
- pnpm 10.27.0（packageManager）；node 24（.node-version + action `using: node24`）

## 12. Snapshot 结论（用于阶段④聚焦）
- **本项目实质**：不是候选卡描述的"纯静态 GitHub Actions 审查 bot"，而是**agent 编排 meta-harness**——在 GitHub Actions 事件驱动下，按配置选择并包装 Claude Code/Codex/OpenCode 执行代码审查/修复/triage，含完整 MCP 工具面、权限边界、凭证生命周期、hook 机制与对抗性安全测试
- **考古重点**：①agent 选择与包装（external.ts / agents/）；②事件→模式→运行编排（main.ts / modes.ts）；③权限模型（push/shell/token/masking/隔离）；④MCP GitHub 工具面（mcp/）；⑤hook 生命周期；⑥模型目录与 effort 映射（models.ts / effort.ts）；⑦失败处理（providerErrors / billingErrors / errorReport / isTransientNetworkError）；⑧对抗测试揭示的安全设计
