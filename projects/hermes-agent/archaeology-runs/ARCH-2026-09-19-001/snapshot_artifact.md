# Repository Snapshot Artifact — ARCH-2026-09-19-001

## 快照事实
| 字段 | 值 |
|---|---|
| repository | https://github.com/NousResearch/hermes-agent.git |
| commit SHA | `fba4cb1cb3f4b65b1528fe9db8ed30aec0c69cf6` |
| commit message | `fix: cache_only Ollama Cloud read no longer rewrites the disk cache` |
| commit timestamp | 2026-09-18 11:01:52 -0700（UTC+8：2026-09-19 02:01） |
| branch | main（浅克隆 depth=1） |
| repository version | v0.21.3（pyproject.toml `version = "0.21.3"`；package.json 为 1.0.0 工作区占位） |
| skill version | knowledge-archaeology v3.2 |
| analysis timestamp | 2026-09-19（日度 SOP） |
| 文件数 | 13,988（git ls-files） |
| 语言 | Python 6,665 / TypeScript 2,406 / Markdown 1,657 / TSX 994 / YAML 302 |

## 项目基础地图
```
hermes-agent/
├── cli.py / run_agent.py / hermes_bootstrap.py   # 入口（CLI/agent 启动/bootstrap 前置导入）
├── hermes_cli/            # CLI 实现（mixins：setup/commands/billing/loops/info/terminal/modal）
├── agent/                 # agent 核心（agent_init / prompt_builder / memory_manager /
│                          #   memory_provider / curator / learning_graph / learning_mutations /
│                          #   context_compressor / conversation_compression / file_safety /
│                          #   credential_pool / skill_commands / skill_utils / background_review）
├── tools/                 # 工具注册表 + 45+ 工具族（registry / memory_tool / skill_manager_tool /
│                          #   skill_linter / skill_manager_guards / skills_guard / checkpoint_manager /
│                          #   approval* / browser_tool* / file_tools_write_guards / hook_output_spill）
├── hermes_state*.py       # SQLite 状态存储族（state / wal / guard / readpool / rewind / repair /
│                          #   compression / sessions / fts / registry / schema / errors / holders）
├── skills/ optional-skills/ # 内置技能目录（含 AGENTS.md、index-cache）
├── plugins/ plugin-catalog/ # 插件 + 插件目录（memory/ 插件族、hermes-memory-*.yaml）
├── providers/             # LLM 提供商适配（anthropic/bedrock/azure/...）
├── gateway/ tui_gateway/ web/ ui-tui/ apps/  # 多前端（消息网关/TUI/Web/Electron desktop）
├── cron/                  # 定时任务
├── evals/ mini_swe_runner.py batch_runner.py trajectory_compressor.py  # 评测/批量/轨迹压缩
├── tests/ (4,645 文件) tests-js/  # 测试
├── AGENTS.md SECURITY.md SOUL.md COMPAT_MANIFEST.md  # 治理与人格
├── pyproject.toml（35 直接依赖 + 45 optional-deps groups）
└── Dockerfile docker-compose*.yml setup-hermes.sh
```

## 识别结果（全部可回溯）
- **主要语言**：Python（6,665 文件）＋ TS/TSX（前端 3,400）
- **主要运行入口**：`hermes` / `hermes-agent` / `hermes-acp`（pyproject `[project.scripts]`）；`cli.py`（CLI，首 import 必为 `hermes_bootstrap`）；`run_agent.py`
- **核心模块**：
  - 状态存储：`hermes_state*.py`（SQLite WAL + FTS5 + readpool + rewind + repair + compression）
  - 记忆系统：`agent/memory_manager.py`（provider fan-out）、`agent/memory_provider.py`（插件 ABC）、`tools/memory_tool.py`（MEMORY.md/USER.md 冻结快照）、`agent/learning_graph.py`/`learning_mutations.py`（§ 分隔卡片）
  - 技能系统：`tools/skill_manager_tool.py`（程序性记忆）、`tools/skills_guard.py`（供应链扫描）、`tools/skill_linter.py`、`agent/curator.py`（空闲维护）
  - 提示构建：`agent/prompt_builder.py`（含 context 注入扫描 `_scan_context_content`）
  - 工具注册：`tools/registry.py`（`registry.register()` 自动发现）
- **核心数据结构**：MEMORY.md / USER.md（§ 分隔 memory:source:index 卡片）、SKILL.md（frontmatter+Markdown+YAML）、state.db（SQLite WAL+FTS5）、.curator_state（JSON）
- **核心状态**：会话（parent_session_id 链压缩）、记忆冻结快照（session 起始）、skills 生命周期（active/pinned/archived）、.curator_state 调度状态
- **主要测试体系**：tests/ 4,645 文件（Python）；tests-js/（TS）；`batch_runner.py`/`mini_swe_runner.py`（RL/benchmark 轨迹）；evals/（memory/ 等评测）
- **主要配置**：`cli-config.yaml.example`、HERMES_HOME 环境（profile-aware）、config.yaml（skills.create_dir / checkpoints / memory.provider）
- **权限/治理机制**：SECURITY.md §2.2 **唯一承重边界=OS 级隔离**（terminal 容器/云沙箱/远程后端）；in-process 启发式（注入扫描/file_safety）明确**非边界**；skills_guard 信任策略（builtin/trusted/community/agent-created）；file_tools_write_guards 保护指令审批；approval* 族（审批门）
- **外部依赖**：35 直接依赖 + 45 optional groups（anthropic/bedrock/azure/mcp/supermemory/mem0/honcho/...）
- **设计约束（AGENTS.md 两不变量）**：① per-conversation prompt caching is sacred（唯一例外 context compression；mutating slash 命令默认 deferred 下 session + opt-in `--now`）；② core is a narrow waist（新能力走 CLI+skill/service-gated/plugin）
