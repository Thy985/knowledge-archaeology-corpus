# 01 Project Layer — dsh-memory-evolve 项目地图

> 全部事实来自 999a0ed 快照实际内容（`repo/`）。lib/ 为权威执行面（测试直接 import，无混淆）。

## 1. 项目身份

| 项 | 值 | 证据 |
|----|-----|------|
| name | dsh-memory-evolve | package.json |
| version | 0.1.0 | package.json |
| 定位 | DSH 长期记忆 + 自我进化 + 技能/待办管理 + CLI 调度 + WebUI | package.json description |
| license | MIT（private: true） | package.json |
| main | lib/index.js | package.json |
| DSH 集成 | dsh.client.inject=[@deepseek-ai/dsh-client-runtime] platform=web + dsh.bundle.patch=cordis.patch.yml | package.json |
| 测试 | node --test 'tests/*.test.js' | package.json scripts |
| 依赖 | 零运行时依赖（node:fs/node:child_process/node:crypto only） | lib/*.js 头部注释 |
| 规模 | lib/ 50,144 行 JS + src/ 21,737 行 TS/TSX | wc |

## 2. 双轨架构（构建期双面）

```
package（dsh plugin add 安装）
├── lib/          ← 后端权威实现（纯 JS ESM，node --test 直接测）
│   ├── index.js（2265 行：插件入口/系统提示注入/工具注册/命令注册）
│   ├── store.js（1444 行：记忆存储层——五轨/锁/原子写/漂移守卫）
│   ├── review.js（回合内记忆审查节拍器 + 建议队列）
│   ├── skills.js + skills-manager.js（技能治理 + 管理）
│   ├── session-orch.js（1355 行：会话编排 spawn/wake/status）
│   ├── sync/（7 文件：identity/repo/merge/worker/filesets/entryid/index）
│   ├── coi/（16 文件：会话协作编排——broadcast/ws-coord/presence/scheduler/…）
│   ├── advisor/（13 文件：隐形评审员——guard/conversation/kinds/scopes/…）
│   ├── search/ + search-docs.js（本地文件搜索：mdfind/find/ripgrep）
│   ├── canvas.js / todo.js / api.js / prompts.js / update.js / notify.js / i18n.js / models.js / bookmarks.js / aliases.js / ui-settings.js / memory-tab.js / coi/…
├── src/client/   ← 前端（TS/TSX React 组件，build.mjs 打包成 lib/client.js 18328 行）
│   ├── index.ts（入口）+ 20+ View 组件（MemoryTab/SkillsTab/TodosTab/SyncView/ModelsTab/…）
│   ├── canvas-grok/（无限画板 10 文件）+ skills-browser/ + advisor/
│   └── 样式/工具（mermaid-render/prompt-styles/mobile/notification-bell/…）
├── tests/         ← 60 个测试文件（node --test，直接 import lib/）
├── scripts/       ← build.mjs（前端打包）+ sync-worker.mjs（同步工作进程）
├── skills/        ← 外部 AI CLI 派单技能（codex/grok/hermes/kimi-cli-calling）
├── vendor/        ← mermaid.min.js（本地 vendor）
├── cordis.patch.yml（host 插件行 bundle 自动注入）
└── docs/（记忆同步.md/CHANGELOG.md）+ README{,.en,-详细说明}.md
```

## 3. 模块面

| 面 | 文件 | 职责 |
|----|------|------|
| **记忆存储** | store.js + sync/entryid.js | 五轨记忆、条目格式、锁/原子写/漂移守卫、ID 生成 |
| **自我审查** | review.js | 回合内记忆审查节拍器（reviewInterval）+ 建议队列（SUGGESTIONS.jsonl） |
| **技能治理** | skills.js + skills-manager.js | skill_manage 工具（create/patch/list/disable）+ read-before-write |
| **待办** | todo.js | 四轨待办（生活/工作/项目/每日）+ 确认队列 |
| **会话编排** | session-orch.js | de_session：spawn/wake/status（标准 DSH 会话） |
| **协作广播** | coi/*（16） | 房间/广播/存在感/调度/ws-coord 冲突协调/attachments/templates |
| **记忆同步** | sync/*（7） | 项目身份/远端仓库/三路合并/worker/文件集/条目 ID |
| **隐形评审** | advisor/*（13） | 会话评审员（note 提取/发射闸门/投递/scope/visible-surface） |
| **本地搜索** | search-docs.js + search/ | 文件名/内容检索（mdfind/find/ripgrep/控制权） |
| **外部派单** | skills/{codex,grok,hermes,kimi}-cli-calling | 统一调度外部 AI CLI |
| **前端** | src/client/*（~50 TSX） | WebUI：记忆/技能/待办/画板/同步/模型/书签/评审/设置 |
| **更新** | update.js | 自我更新（tag 检查/dirty 拦截/退避/信任校验/API 契约） |

## 4. 核心数据结构

| 结构 | 定义 | 语义 |
|------|------|------|
| 记忆条目 | `
§
` 分隔纯文本（Hermes 字节兼容） | MEMORY.md/USER.md 格式 |
| 五轨记忆 | 用户档案(USER.md) / 全局事实(MEMORY.md) / 项目关键记忆 / 项目日志 / 每日日志 | 分层注入上下文 |
| 学习轨道 | SUGGESTIONS.jsonl | 建议队列（用户确认才转正） |
| 待办 | TodoStore（四轨） | 生活/工作/项目（按 cwd 隔离）/每日 |
| 项目身份 | normalizeRemoteUrl → `host[:port]/path` 键 | 跨协议收敛（https/ssh 同键） |
| 同步文件集 | filesets.js | 各轨独立开关的同步文件集合 |
| 评审 note | advisor 管道（note→guard→delivery） | 隐形评审员输出 |
| 冲突 | 内存三路合并（base/ours/theirs）+ 冲突清单 | git 冲突标记永不落盘 |

## 5. 核心状态

- **记忆状态**：五轨文件（$EVOLVER_HOME 或项目目录）+ 每日文件（todayStamp 命名）
- **审查状态**：每会话 turn 计数器（agent/turn-stopping）+ reviewInterval 触发 DUE；**计数器不自动重置**（memory_review_status complete 才重置）
- **锁状态**：跨进程锁文件（STALE_LOCK_MS=10s 视为废弃；LOCK_TIMEOUT_MS=5s 超时 loud fail；自旋 25ms）
- **同步状态**：身份键 → 远端分支（shared/legacy/shared-fresh/专属）；PROVENANCE 归属标记
- **ws-coord 状态**：资源锁（TTL 默认 1min/上限 5min，写操作自动续期；observed 锁回合结束不释放）+ 占用集
- **更新状态**：localTag/outdated/restartRequired/退避（30min）/损坏备份 .corrupt-*

## 6. 测试体系（60 文件 / 798 断言——主题化命名）

| 主题 | 代表文件 | 实测 |
|------|---------|------|
| 记忆存储 | store.test.js（parseEntries/serializeEntries/isCanonical/漂移守卫） | ✅ |
| 同步 | sync-*.test.js（18 文件：identity/merge/repo/worker/conflict/e2e/global/shared-branch） | ✅（git 环境配置后） |
| 评审 | advisor-*.test.js（11 文件）+ review-*.test.js | ✅ |
| 会话编排 | session-orch.test.js | ✅ |
| 协作 | coi.test.js / ws-coord / broadcast-image | ✅ |
| 更新 | update.test.js（47 断言：锁/退避/信任/dirty/API 契约） | ✅（git 环境配置后） |
| 技能 | skills.test.js / skills-manager.test.js | ✅ |
| i18n | i18n.test.js（中文契约锁定）+ review-i18n | ✅ |
| 前端契约 | memory-tab / todo-tab-lifecycle / progressive-disclosure / ui-settings | ✅ |
| 其他 | api/bookmarks/canvas/mermaid/models/notify/prompts/search-docs/session-search/store-summary | 部分 |

**环境契约三件套（本 run 实测发现）**：①git init 默认分支必须 main（update.test.js setupRepo 硬编码 `git push origin main`）②git user 身份必须配置（commit 失败）③search-docs 硬编码 darwin（mdfind）预期。配置后 798 → 795 pass / **3 fail 全为 darwin 平台假设**。

## 7. 主要配置

| 配置 | 默认 | 用途 |
|------|------|------|
| reviewEnabled | false | 回合内记忆审查开关 |
| reviewInterval | 10 | 每 N 用户回合审查一次 |
| 记忆目录 | $EVOLVER_HOME（~/.evolver 风格）+ 项目目录 | 五轨文件位置 |
| skills 目录 | ~/.agents/skills | 共享技能目录（DSH skill-local + Hermes 外部目录都扫描） |
| 同步开关 | 各轨独立 | 全局/每日/项目/待办四轨独立开关 |
| ws-coord | 默认关 | 工作区冲突协调（软模式） |
| 同步远端 | 代码仓库（专属分支）/ 共享记忆仓库 | 记忆存储位置 |

## 8. 权限 / Policy / Governance 机制

| 机制 | 证据 |
|------|------|
| **用户确认门** | user track 只由显式 memory 工具调用或用户确认的建议写入；learned track 建议 → /memory_review 确认才转正 |
| **read-before-write** | skill_manage patch 要求会话日志含 skill_manage action=read 事件（证据链） |
| **路径安全** | skill 名必须 kebab-case（规则化排除路径穿越）；fetch --out= 遏制（若沿用） |
| **漂移守卫** | 全文件重写前必须 round-trip 解析通过；漂移文件备份 .bak.<ts> 后拒绝 |
| **冲突不落盘** | 三路合并内存决策；双侧改不同 → 人工处理队列 |
| **软协作** | ws-coord 默认软模式（警告不拦截）+ fs/observed 自动登记（硬证据） |
| **评审闸门** | guard：NFKC 归一化 + 空泛短语抑制 + 每轮一条 + 去重升级 |
| **更新安全** | dirty 拦截（untracked/staged）/退避 30min/信任校验（tag 须在 origin/main 历史）/API 同源校验 |

## 9. 主要外部依赖

- **零运行时依赖**（唯一例外：可选系统命令 mdfind/find/ripgrep、git、外部 AI CLI codex/grok/hermes/kimi）
- **构建期**：esbuild（从 DSH source checkout 解析，不打包进插件）
- **宿主**：@deepseek-ai/dsh-client-runtime（client inject）+ DSH core public seams（systemPrompt/tools/commands/subagents/approval）
