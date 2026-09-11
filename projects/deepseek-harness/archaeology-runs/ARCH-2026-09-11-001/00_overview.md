# 00 — Overview：DeepSeek Harness v0.1.5-rc.2（REFRESH）

## 一句话定位

DeepSeek Harness（`dsh`）是 DeepSeek AI 开源的 **everything-is-a-plugin** Agent Harness（Cordis 内核）：事件溯源的 Session 日志是唯一事实源，工具执行经 sandbox/approval/guard 分层权限管线，能力以 Capability Seam 可插拔，PTC 模式让模型写 TypeScript 而非逐个 tool-call。

## 本次考古性质

- **模式：REFRESH**（job ARCH-2026-09-11-001）
- 基线：ARCH-2026-09-03-001（v0.1.2-rc.1 / 76fda72 / 32 EK / 8 KO / 9 candidates）
- 目标：v0.1.5-rc.2 / fb2c4b9

## 本轮三个核心发现（Top Findings）

### F1 — 09-03 考古存在整块 Coverage Gap：`.agents/notes` ADR 体系被遗漏
09-03 考古包 32 条 EK 对 `.agents/notes` 的引用为 **0**，而该 commit（76fda72）已存在 862 条（现 958 条）结构化 Agent Notes（architecture/feature/bug-fix/process/simplification/testing 六类）。这些笔记是本项目最完整的"决策证据源"——回答"为什么这样做"，恰是 Knowledge Archaeology 的核心命题。本轮以其为主证据源补 EK。

### F2 — PTC 从设计到执行的完整演化链（跨 3 个月，5 篇 notes）
- 06-15 原始设计：模型对工具注册表写 TypeScript → `run_code` 传输（Cloudflare Code Mode 启发），worker_threads 沙箱；
- 07-10 并行调度、07-20 typed returns、07-26 live parallel dispatch；
- **08-07 executor collapse 修复**：wire 只通告 `run_code` 但 executor 曾放行全部工具（schema 省略≠强制），改为 `resolveExecution(name,scope,nested)` 在 `createExecution` 处（**先于策略管线**）确定性折叠为 UNKNOWN_TOOL——实证了"拒绝必须经 executor 测试"的包契约；
- 08-25 命名纪律：code-mode → ptc 全量迁移（config/preset/文件），无兼容别名。
→ 验证 09-03 EK-28（PTC collapse 隐藏契约）并补全其因果链。

### F3 — 权限边界的设计演进：workspace-files（09-05）被 09-09 显式超驰（supersede）
- 09-05 初版：文件方法 workspace-contained（第二层读策略）；
- 09-09 修订：`read/readBytes/readAll/readRelated/stat` **继承 Session fs 后端的读授权**，workspace 根只是相对路径基准而非读边界；`list/changes` 仍 workspace-scoped；`readRelated` 允许 `..` 外读 JS/CSS；Document Preview 用 `sandbox="allow-scripts"` opaque-origin iframe 渲染静态 HTML——**网络访问保留是有意安全取舍，并明确记录 Alternatives considered**。
→ 这是"Design Intent vs Implementation Reality"与权限分层治理的活案例。

## 雷达声明 vs 仓库证据（证据等级）

| 雷达/媒体声明 | 仓库证据 | 判定 |
|---|---|---|
| v0.1.5 发布（2026-09-10） | tag dsh-v0.1.5-rc.2、pushed_at 2026-09-10T15:06:56Z、version 0.1.5-rc.2 | 已查证 |
| 文件上传/侧栏预览 | workspace-files-service（09-05）、sidebar/preview notes（09-07~09-10）、document-preview-operations（09-08） | 已查证（notes 体系） |
| 标准/PTC/极简三模式联合训练 | PTC mode 存在（presets/ptc、config tools.mode:'ptc'）；"三模式联合训练"（模型×Harness）在仓库内无直接证据 | PTC 模式已查证；联合训练 = 一方称 |
| 181 插件生态 | everything-is-a-plugin + 55 个 packages；无 181 的仓库内清单 | 55 packages 已查证；181 = 一方称 |

## 09-03 基线主张在 v0.1.5 的存续性（摘要）

- **仍成立（验证/深化）**：事件溯源日志唯一事实源（V3 envelope 强化）、Capability Seam、approval fail-closed、sandbox 三态、结构化错误族、Cordis 插件内核、PTC collapse、权限 fail-closed 不变量族、日志即真相。
- **已演进**：会话格式 V2→V3（canonical envelopes，2026-09-06）；permission 边界新增 workspace-file-read-authority 分界；新增 python SDK 测试面（S4）。
- **无被推翻的基线主张**（本轮未发现 baseline EK/KO 被 v0.1.5 反证）。

## 质量指标速览（详见 06）

- EK：32 基线核对 + **13 条新增**（EK-R01..R13）→ 45 条，全部声明 links（EK Graph）
- KO：8 基线核对 + 2 新增（KO-09、KO-10）→ 10 个
- Candidates：5 新增（含跨项目假说）
- Flow：7 类基线 + PTC collapse 路径补充
- 测试证据：python SDK **111 passed / 7 skipped**（S4 实测）
- 独立验证：ACCEPT（详见 06）
