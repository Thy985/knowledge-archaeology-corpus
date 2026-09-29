# Snapshot Artifact — ARCH-2026-09-30-001

## 仓库快照
| 项 | 值 |
|---|---|
| repository | https://github.com/microsoft/Webwright.git |
| commit | `bc26750af3ad166d982f23d101ef8971a3a2fce5` |
| commit_message | "Add Web Skill Factory: self-evolving library of verified, code-native web skills (#62)" |
| branch | main |
| commit_date | 2026-08-03 15:00:26 -0700 |
| clone | `git clone --depth 1`（2026-09-30 本地） |
| version | 0.1.0（pyproject.toml） |
| license | MIT（LICENSE） |
| language | Python ≥3.10 |
| tracked_files | 135 |

## 项目基础地图

### 定位（README 原文）
"Webwright gives LLM a terminal where it can launch multiple browser sessions to inspect the page and complete a web task… your web agent browsing history is a single code file. No multi-agent system, no graph engine, no plugin layer, no hidden orchestration — just a terminal, a browser, and a model."
范式：**coding agent with a terminal**——浏览器是可启动/检查/丢弃的环境；持久产物是**本地工作区中的代码与日志**（workspace-as-state, not browser-as-state）。与 Stagehand / agent-browser / browser-use 的"browser session 即 state"范式相反（README 对比表）。

### 入口
- CLI：`webwright = webwright.run.cli:app`（pyproject scripts）；`python -m webwright.run.cli main -t <task> --start-url <url> -o <out> --task-id <id> [-c base.yaml -c model_*.yaml]`
- 工具入口：`python -m webwright.tools.image_qa` / `webwright.tools.self_reflection` / `webwright.tools.skill_use`
- Skill Factory 入口：`python -m webwright.skill_factory <init|build|learn|update>`（__main__.py）
- 宿主插件：`.claude-plugin/plugin.json`（Claude Code）、`.codex-plugin/plugin.json`（Codex）；`skills/webwright/` 跨 Claude Code / Codex / OpenClaw / Hermes 四宿主加载
- 环境诊断：`python -m webwright.run.doctor`

### 模块地图（src/webwright/）
| 模块 | 行数 | 职责 |
|---|---|---|
| agents/default.py | 467 | DefaultAgent：agent loop、Self-Reflection Gate、Tool Gate、ARIA 裁剪、LLM 压缩、debug 工件 |
| environments/local_workspace.py | 296 | 无状态工作区 harness：bash 执行、凭证 env、步骤日志、观测捕获 |
| environments/local_browser.py | 567 | 实时浏览器 harness：CDP/持久化/本地启动、python_code 执行、ARIA 快照、控制台捕获 |
| models/base.py + anthropic/openai/openrouter | ~600 | 后端无关模型协议：strict JSON、重试/退避、usage 指标、key 校验 |
| run/cli.py | 175 | typer CLI：配置栈合并、输出目录、task 校验 |
| run/doctor.py | 147 | 环境诊断（python/playwright/chromium/…） |
| tools/image_qa.py | — | 截图问答（视觉检查工具） |
| tools/self_reflection.py | ~500 | 两阶段图片判定器（per-image Score + final Status verdict） |
| tools/skill_use.py | — | skill 使用工具 |
| tools/persistent_local_browser.py | — | 持久浏览器工具 |
| skill_factory/（15 模块） | ~3000 | init/build/learn/update：把 solve 蒸馏为可复用参数化 code skill |
| config/（9 yaml） | — | base（workspace 模式）+ model_* + local_browser + persistent_browser + task_showcase + crafted_cli |
| exceptions.py | 24 | InterruptAgentFlow / LimitsExceeded / Submitted / FormatError |

src 总计 8061 行（含 skill_factory）；核心 agent loop 467 行、Playwright env 567 行、CLI 175 行（README 宣称 ~1.5k LoC 核心，为 skill_factory 合并前规模）。

### 核心数据结构
- **动作协议**：单 JSON 对象 `{thought: str, <action_field>: str, done: bool, final_response: str}`；strict schema（additionalProperties: False，required 四字段，models/base.py `_response_schema`）。action_field 因模式而异：workspace=`bash_command`、live browser=`python_code`。
- **观测**：`{success, workspace_dir, cwd, command, returncode, exception, command_output, final_script_path}`（workspace 模板）/ `{success, url, title, exception, python_output, console_output, aria_snapshot, screenshot_path}`（browser 模板）。
- **Skill 单元**：目录 = `skill.py + meta.json + replays.json`（library.py：`Skill`/`Library`）。skill.py 头部由 entry_shim 自动注入确定性 CLI。
- **判定产物**：`self_reflect_result.json` = `{per-image records(Score/Reasoning/Response), image list, final prompt, final response, predicted_label: 1|0|null}`。

### 状态模型（双模式）
- **workspace 模式（base.yaml）**：状态在磁盘——`workspace_dir/{task.json, plan.md, self_reflect_config.json, final_script.py, final_runs/run_<id>/{final_script.py, final_script_log.txt, screenshots/*.png}, trajectory.json}`。无持久浏览器状态；每步新开浏览器会话。
- **live browser 模式（local_browser.yaml）**：状态在浏览器——page/context/browser/playwright 跨步持久；无 workspace、无 final_script、无 self_reflection（`require_self_reflection_success: false`）。

### 配置要点
| 键 | 默认值 | 语义 |
|---|---|---|
| agent.require_self_reflection_success | true | done=true 需 self_reflect_result.json predicted_label==1（workspace 模式） |
| agent.summary_every_n_steps | 20 | LLM 压缩历史频率 |
| agent.step_limit | 100 | 步数上限（LimitsExceeded） |
| model.keep_last_n_observations | 1（browser）/ 0（workspace） | ARIA 快照保留数（token 治理） |
| environment.command_timeout_seconds | 240 | bash 命令超时 |
| model.max_output_tokens | 4000 | 单响应输出上限（大 skill 复用时需 16000，见 agent_cfg 注释） |
| model.attach_observation_screenshot | false | 截图默认不自动附 prompt（省 token） |
| run 输出 | outputs/default | trajectory.json 每步落盘 |

### 测试体系
- 14 个测试文件、1899 行、全部 **LLM-free**（纯函数/桩）。
- 覆盖：gate（准入）、route（run/adapt/skip/fallback）、recommend（promote）、fill（槽填充）、execute、learn、library、entry_shim、evolve、retrieve/decide、doctor、tool_model_routing、learned_example。
- CI：`.github/workflows/skills-tests.yml`——push/PR 时**仅当路径命中** `src/webwright/skill_factory/**`、`src/webwright/tools/skill_use.py`、`tests/skill_factory/**` 才触发；`PYTHONPATH=src python tests/skill_factory/test_*.py` 逐个直跑；F6 wrapper usage check（缺参必须 exit 1）。

### 权限与治理机制
- 无内部权限模型（非 Agent 安全项目）；治理体现在 **Completion Gate**（base.yaml 五条硬清单：plan.md、self_reflect_config.json、final_script 从零执行、self_reflection exit 0 + predicted_label==1、ls/cat 工件确认）与 **Task Success Criteria**（8 条：专用控件必须用、数值精确匹配、ranking 用语必须落站点真实指标等）。
- 凭据：模型 API key（OPENAI_API_KEY / ANTHROPIC_API_KEY）与 Browserbase（BROWSERBASE_API_KEY/PROJECT_ID）；key 缺失 → RuntimeError（models/base.py）；serialize 时 key 脱敏为 `<redacted>`。

### 外部依赖
httpx / jinja2 / pydantic / pyyaml / rich / typer / playwright / python-dotenv / platformdirs；模型后端 OpenAI / Anthropic / OpenRouter；可选 Browserbase 云会话。

### 版本/时间线（README News + git log）
- 2026-05-04 初始发布（~1.5k LoC，OpenAI/Anthropic/OpenRouter，Playwright env）
- 2026-05-06 Codex/Claude Code 插件 + OpenClaw/Hermes 集成
- 2026-05-11 Task2UI 模式（任务结果渲染为 HTML web app）
- 2026-07-21 **Skill Factory**：solve 留脚本 → 蒸馏为可复用已验证参数化 code skill（无模型重跑 ~40s zero tokens；WebArena reuse 55%→70% +15pp）
- 2026-08-03 HEAD（Skill Factory 模块化重构，PR #62）
