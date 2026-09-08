# 00 · Overview — OpenCode（ARCH-2026-09-09-001）

> 核心问题：**"这个项目让我们认识到了什么？"** —— 一个曾增长最快的开源 agent coding harness（Charm 团队），如何以 Go 单体 TUI 架构构建工具执行、权限批准与 LSP 集成，又为何在巅峰归档、迁移到 Crush。

## 一句话定位

OpenCode 是 Go 编写的终端 AI coding agent（Bubble Tea TUI + SQLite 持久化 + LSP 集成 + 多 LLM provider），Charm 团队早期 harness 单体 TUI 路线代表。**本仓库已于 2025-09-17 归档**（archived: True），项目继续以 [charmbracelet/crush](https://github.com/charmbracelet/crush) 名义开发。

## ⚠️ 候选卡事实修正（Source Fidelity 关键）

KnowlegeMap 雷达 #8 候选卡 [cand]opencode-agent-2026 描述"**client/server 解耦 headless Hono HTTP 服务 + Vercel AI SDK、~147k★、650 万月活、GitHub Copilot 官方合作**"——**与本仓库实际内容严重不符**（一手核验 2026-09-09）：

| 项 | 候选卡声称 | 仓库实际（verified） |
|----|-----------|---------------------|
| 架构 | headless Hono HTTP + Vercel AI SDK | **Go 单体 TUI**（Bubble Tea），无 TS/Hono 文件 |
| star | ~147k | **13,727**（GitHub API） |
| 状态 | 活跃 | **已归档**（2025-09-17，迁移 charmbracelet/crush） |
| Copilot | 官方合作 | 代码含 copilot provider（认证直连），但无"合作"证据 |

**处理**：考古以仓库实际内容为准；事实偏差记录于本文件与 05 Candidates，不改 KnowlegeMap（只读）。

## 为什么值得考古（对个人知识系统的价值）

1. **harness 架构谱系对照**：与已考古 DeepSeek Harness（validated）同赛道不同路线——单体 TUI（Bubble Tea）vs 单体 CLI；Charm 团队演进到 Crush 的设计演化故事。
2. **工具执行 + 权限批准模型**：所有副作用工具（bash/edit/write/patch/fetch）统一走 permission.Service 门；会话级 AutoApproveSession 信任提升；同步阻塞审批（无超时）——与 omnigent ASK（86400s 超时）形成审批设计对照。
3. **LSP 深度集成**：protocol 生成代码 6907+3072 行 + watcher 文件变更跟踪 + diagnostics——编辑器智能（类型错误）直接进 agent 上下文。
4. **失败与修复证据**：agent.go:373 "Monkey patch for Copilot Sonnet-4 tool repetition obfuscation"（模型重复调用修复）；测试揭示 WorkingDirectory panic bug。
5. **早期项目的诚实画像**：4 个测试文件 vs 42k 行代码、panic-instead-of-error、归档决策——"增长最快"与"工程成熟度"的解耦。

## 关键数字（全部可追溯到仓库）

| 项 | 值 | 来源 |
|----|-----|------|
| 规模 | 140 go 文件 / 42,162 行 | find + wc |
| 语言 | Go 1.24.0（module） | go.mod |
| HEAD | 73ee493265acf15fcd8caab2bc8cd3bd375b63cb | git rev-parse |
| 归档 | 2025-09-17（archived: True，pushed_at 2025-09-18T02:54:28Z） | GitHub API |
| stars | 13,727 | GitHub API |
| 测试 | 4 个 *_test.go（prompt/ls/theme/custom_commands） | find |
| 测试运行 | 3 包 ok + 1 包 FAIL（ls panic） | go test 实测（S4） |
| 编译 | 通过（GOTOOLCHAIN=auto go1.24） | go build 实测 |

## 产物清单

| 文件 | 内容 |
|------|------|
| 01_project-layer.md | 项目地图 |
| 02_engineering-knowledge.md | EK Graph（27 EK + 六类边） |
| 03_knowledge-layer.md | Generalized Knowledge（6 KO + 4 CM + 3 M） |
| 04_flow-atlas.md | 七类流 |
| 05_candidates.md | 未验证假说与跨项目候选 |
| 06_validation.md | 验证 + 质量指标 |
| run_metadata.yaml | run 元数据 |

## 免责声明

- 快照基于浅克隆 HEAD 73ee493（归档版 final commit）。
- 测试运行画像：`go build` 通过；`go test` 实测 3 包 ok（prompt/theme/dialog）+ 1 包 FAIL（tools：ls_test panic WorkingDirectory）——S4 证据有限但真实，非全量测试（未运行包多数无测试文件）。
- 候选卡事实偏差已修正（见上），不影响考古本体。
