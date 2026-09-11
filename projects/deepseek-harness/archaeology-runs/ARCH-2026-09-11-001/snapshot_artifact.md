# Repository Snapshot Artifact — deepseek-harness v0.1.5-rc.2

## Snapshot

| 字段 | 值 |
|---|---|
| repository | https://github.com/deepseek-ai/deepseek-harness.git |
| commit_sha | `fb2c4b9e698e30edb738bca4cf0618587db7d203` |
| branch / tag | master @ tag `dsh-v0.1.5-rc.2` |
| repository_version | `0.1.5-rc.2`（package.json `@deepseek-ai/dsh-root`） |
| analysis_timestamp | 2026-09-11T02:00+08:00（cron 触发） |
| shallow_clone | `--depth 1 --branch dsh-v0.1.5-rc.2` → `archaeology-jobs/ARCH-2026-09-11-001/repo` |
| 规模 | 10178 files / 121MB / 3287 个 .ts（本地实测） |
| 基线（09-03 考古） | ARCH-2026-09-03-001 / v0.1.2-rc.1 / `76fda729799fe9b3848dbe2c211d4b231032b81e`（本轮 fetch 成功，可精确 diff） |
| skill_version | knowledge-archaeology v3.2（本地生产版，未修改） |
| 凭证 | /home/user/.git-credentials（Thy985），三仓库 fetch 均 OK；corpus push 权限待阶段 6 实测 |

## 项目基础地图（事实均来自仓库实际内容）

```
deepseek-harness/
├── packages/            # 55 个 Cordis 插件包（everything-is-a-plugin 内核）
│   ├── core/            #   核心：session/surface、事件溯源、代理循环
│   ├── api/             #   workspace-files（v0.1.5 新）/ gateway / remotes
│   ├── sandbox/         #   沙箱：landlock/bwrap/ACL 受限 token（Windows）
│   ├── guard/           #   守卫：单调拒绝、approval、repeat-tool-reminder
│   ├── llm/             #   LLM adapter 注册表（twin adapters / provider-routed）
│   ├── subagent/        #   子代理委托 seam（多 provider / policy 继承）
│   ├── acp/             #   ACP 自动化协议桥（automation-only）
│   ├── code-runtime/    #   执行世界：PTC run_code / python fd3 协议
│   ├── e2b/             #   E2B 远端执行适配
│   ├── session/ session-query/ session-persistence/ session-log  # 会话家族
│   ├── feedback/        #   规范化反馈日志 + OTEL（v0.1.5 新）
│   ├── workspace/ web/ client/  # 客户端与 web 侧栏（v0.1.5 文件服务/预览）
│   └── …（acp/attachment/boot/bundle/compaction/context/credentials/experimental/
│        extensions/fs/goal/hooks/host/identity/interaction/jobs/lsp/mcp/plan/
│        preset/runtime-diagnostics/schedule/sdk/settings/shell/skill/spill/
│        storage/terminal/test-support/todo/typert/util/webhook/workflow）
├── apps/                # 应用壳（web / electron desktop）
├── python/              # v0.1.5 新：Python SDK（subprocess JSON-RPC over stdio）
│   ├── sdk/             #   deepseek-harness-sdk（111 passed / 7 skipped）
│   └── sdk-runtime/     #   捆绑 dsh CLI 运行时 wheel
├── docs/                # 用户指南 + architecture.md
├── benchmarks/          # benchmark 脚本
├── .agents/notes/       # ★ 958 条 Agent Notes（ADR 体系）——09-03 考古零引用
│   └── implemented/{architecture,feature,bug-fix,process,simplification,testing}/ + archived/
├── native/ vendor/ website/ scripts/ snapshots/
├── SAFETY.md            # 安全声明：experimental、未经审计、沙箱不保证隔离
├── BENCHMARK.md         # Python SDK jsonrpc-agent 最小变体 benchmark 指引
└── pytest.ini           # testpaths = python/sdk/tests（限制根收集）
```

## 主要语言 / 入口 / 核心模块 / 数据结构 / 状态 / 测试 / 配置 / 权限 / 依赖

| 维度 | 事实 | 证据 |
|---|---|---|
| 主要语言 | TypeScript（3287 .ts）+ Python SDK（v0.1.5 新增） | 本地文件统计；python/sdk |
| 主要运行入口 | `npx @deepseek-ai/dsh web`；源码运行 `pnpm dsh web`（默认 http://127.0.0.1:3080） | README.md |
| 核心模块 | packages/core（session surface/事件溯源）、packages/sandbox+guard（权限）、packages/llm（adapter）、packages/subagent、packages/code-runtime（PTC） | packages/ 布局 |
| 核心数据结构 | SessionEvent（append-only JSONL）、Session V3 envelope（surfaceOp append/replace）、ToolCall/Result、WorkspaceFileScope | 02 EK；2026-09-06-v3-canonical-session-envelopes |
| 核心状态 | Session 事件日志（唯一事实源）+ 投影（Projection 增量折叠）；Agent 回合循环 turn/step | 09-03 考古 EK-01/02/05；notes event-sourced-sessions |
| 测试体系 | vitest（packages）+ pytest（python/sdk/tests，**本轮实测 111 passed/7 skipped**）+ GUI e2e + snapshot corpus | pytest 实测；notes testing/* |
| 主要配置 | presets（per-session agent presets）、settings seam（分层解析+脱敏）、config `tools.mode:'ptc'` | 09-03 EK-30；2026-08-25 rename note |
| 权限/治理 | approval（fail-closed + 可重放策略）、sandbox 三态、permission presets、guard 单调拒绝、PTC collapse 先于策略管线、workspace-file-read-authority | 09-03 EK-09..14；本轮 notes |
| 外部依赖 | Cordis 插件内核、LLM providers（DeepSeek 等，provider-routed）、E2B、npm 生态、Node ≥ 某版本（process note） | 09-03 EK-24；notes |

## 09-03 → v0.1.5 量化差异（git 实测）

| 指标 | 76fda72（09-03） | fb2c4b9（本轮） | Δ |
|---|---|---|---|
| .agents/notes 非翻译 md | 862 | 958 | **+96**（09-05 后加速：workspace-files/sidebar/preview/feedback） |
| packages 顶层 | 50 | 55 | +5（含 api/workspace-files、feedback 等） |
| python/ | 存在 | SDK 完整（uv.lock + 111 tests） | SDK 成型 |
| SAFETY.md / BENCHMARK.md | 存在 | 存在 | 不变（09-03 考古未引用） |

> 注：09-03 考古 EK 对 `.agents/notes` 引用数为 **0**——整块 ADR 证据源被遗漏，本轮 refresh 的首要补偿对象。
