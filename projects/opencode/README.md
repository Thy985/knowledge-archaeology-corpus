# OpenCode — Knowledge Archaeology

Go 终端 AI coding agent（Charm 团队早期单体 TUI harness 路线）：Bubble Tea TUI + 工具权限门 + SQLite 持久化 + LSP 深度集成 + 多 LLM provider（含 Copilot 认证直连）。**2025-09-17 归档**，项目迁移 [charmbracelet/crush](https://github.com/charmbracelet/crush)。

## ⚠️ 候选卡事实修正（重要）

KnowlegeMap 雷达 #8 候选卡描述（headless Hono HTTP + Vercel AI SDK + ~147k★ + 650 万月活）**与本仓库实际严重不符**：GitHub API 实测 13,727★、Go 单体 TUI（无 TS/Hono）、已归档。考古以仓库实际内容为准。

## 考古时间线

| Run | 日期 | 模式 | commit | 结果 |
|-----|------|------|--------|------|
| ARCH-2026-09-09-001 | 2026-09-09 | initial | 73ee493 | ✅ 合并（PR #9） |

## 关键结论（速览）

- **工具副作用统一授权门**：bash/edit/write/patch/fetch 全部注入 permission.Service，Run 前 Request；拒绝 = ErrorPermissionDenied——授权与执行解耦，治理在执行路径上（非模型自觉）
- **审批三层模型**：AutoApproveSession（会话信任提升）→ sessionPermissions 缓存（同参批准）→ pendingRequests 同步阻塞（无超时——与 omnigent ASK 86400s 超时形成设计对照）
- **编辑器智能进 harness**：LSP protocol 生成代码 + watcher 文件变更 + diagnostics 注入 + diff/history 可视化——单体 TUI 路线的结构性差异化
- **harness 记忆**：SQLite + SQLC + goose 持久化会话/消息/文件快照，会话可恢复
- **增长 ≠ 成熟**：13.7k★ 但仅 4 个测试文件；测试实测 3 包 ok + 1 包 FAIL（ls_test panic config.WorkingDirectory config.go:874）；随即归档迁移 Crush

## 产物清单

- `00_overview.md` — 总览（候选卡事实修正 + 认知核心 + 关键数字）
- `01_project-layer.md` — 项目地图
- `02_engineering-knowledge.md` — EK Graph（27 EK，六类边）
- `03_knowledge-layer.md` — 6 KO（R1-R4 聚合）+ 4 CM + 3 M
- `04_flow-atlas.md` — 七类流
- `05_candidates.md` — 8 候选（含 C-04 需人工审查）
- `06_validation.md` — 验证 + 质量指标（真实 S4 测试证据）
- `archaeology-runs/ARCH-2026-09-09-001/` — 原始 run 快照

## 证据说明

- 快照：浅克隆 HEAD 73ee493（归档 final commit，2025-09-17）
- 测试运行（S4 实测）：`go build` 通过；prompt/theme/dialog 3 包 ok + tools FAIL（ls_test panic）
- 开放项：C-02（同步阻塞挂起模式）、C-04（候选卡 headless 描述失真根因——需核对 crush）
