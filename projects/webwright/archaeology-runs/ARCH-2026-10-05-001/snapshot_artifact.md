# Repository Snapshot Artifact — Webwright (microsoft/Webwright)

- **run_id**: ARCH-2026-10-05-001
- **project**: Webwright
- **repository**: https://github.com/microsoft/Webwright.git
- **mode**: initial
- **snapshot time**: 2026-10-05（UTC+8）

## 1. HEAD 事实（可回溯）

| 字段 | 值 |
|---|---|
| commit SHA | `bc26750af3ad166d982f23d101ef8971a3a2fce5` |
| branch | main |
| commit date | 2026-08-03T15:00:26-07:00 |
| commit subject | `Add Web Skill Factory: self-evolving library of verified, code-native web skills (#62)` |
| version | 0.1.0（pyproject `[project] version`；`src/webwright/__init__.py __version__ = "0.1.0"`） |
| requires-python | `>=3.10`（pyproject.toml） |
| license | MIT（LICENSE，Copyright Microsoft Corporation） |
| 浅克隆位置 | `archaeology-jobs/ARCH-2026-10-05-001/repo/`（depth 1，只读） |

## 2. 项目定位（README 原文证据）

> "Give LLM a terminal where it can launch multiple browser sessions… enforces each web task to be completed end-to-end within a re-runnable Python script… No multi-agent system, no graph engine, no plugin layer, no hidden orchestration — just a terminal, a browser, and a model."

对比表（README 四行）：Stagehand / agent-browser / browser-use / Webwright 在 Paradigm / Action space / What is state / Loop shape 上对照——Webwright 的 state 是 **local workspace（code, screenshots, logs）**，browser 是 disposable（可丢弃），loop 是 `write code → execute → inspect screenshots → repair`。

**News 时间线（README）**：
- 2026-05-04 initial release（~1.5k LoC，OpenAI/Anthropic/OpenRouter 后端，Playwright 环境）
- 2026-05-06 Codex/Claude Code 插件 manifest；OpenClaw/Hermes 集成（同一 `skills/webwright/` 目录加载到四宿主）
- 2026-05-11 Task2UI 模式
- 2026-07-21 **Skill Factory**：每个 solve 留下脚本，蒸馏为可复用/已验证/参数化 code skills，standalone 重跑 ~40s zero tokens；WebArena reuse 55%→70%（+15pp）

## 3. 项目基础地图（全部可回溯仓库实际内容）

### 3.1 语言与依赖
- Python ≥3.10；包名 `webwright`；console script `webwright = webwright.run.cli:app`
- 运行时依赖 10 项：httpx≥0.27 / jinja2≥3.1 / pydantic≥2.5 / pyyaml≥6.0 / rich≥13.0 / typer≥0.12 / playwright≥1.45 / python-dotenv≥1.0 / platformdirs≥4.0（pyproject.toml）
- **"zero hidden frameworks"**（README 声明，代码验证）：无 agent 框架/graph engine/plugin layer，自研 provider-agnostic 模型层

### 3.2 源码规模（wc -l 实测，src 总计 6552 行；tests 1899 行）
```
src/webwright/
├── agents/default.py(467)            核心 agent loop（AgentConfig + 自我反思门控）
├── agents/__init__.py                工厂 get_agent（agent_class 映射）
├── environments/local_browser.py(567)  LocalBrowserEnvironment（三 browser_mode）
├── environments/local_workspace.py(296) LocalWorkspaceEnvironment（bash 工作区）
├── environments/__init__.py          工厂 get_environment（默认 local_workspace）
├── models/base.py(587)               provider-agnostic BaseModel（重试/JSON 修复/bash 校验）
├── models/openai_model.py(157)       OpenAI Responses API + json_schema strict
├── models/anthropic_model.py(192)    Anthropic backend
├── models/openrouter_model.py(206)   OpenRouter backend
├── models/__init__.py                工厂 get_model
├── run/cli.py(175)                   typer CLI 入口（run_one 装配/snapshot/debug）
├── run/doctor.py(147)                环境自检（6 项 PASS/FAIL）
├── config/__init__.py(85)            配置解析（path/builtin/inline key=value 点分嵌套）+ snapshot
├── tools/self_reflection.py(611)     两阶段截图裁判（Stage1 每图评分+Stage2 聚合 verdict）
├── tools/persistent_local_browser.py(314) 长活 Chromium 会话管理 CLI（create/info/release）
├── tools/skill_use.py(149)           recommend 决策（retrieve→decide→promote）
├── tools/image_qa.py(141)            截图视觉问答 CLI
├── tools/_model_config.py(77)        工具模型加载（config_snapshot/merged_config.yaml）
├── skill_factory/（2150 行，14 模块）
│   ├── update.py(453)                evolve/refine（蒸馏+重放验证+grade 三态）
│   ├── learn.py(336)                 run 目录→技能（gate→分组→canonicalize→evolve）
│   ├── build.py(236)                 spec→solve N 实例→learn（并行+断点续跑）
│   ├── route.py(217)                 run/adapt/skip 路由（直接跑 or 交给 agent）
│   ├── init.py(166)                  一行需求→skill.yaml 草稿
│   ├── decide.py(114)                use/adapt/skip 判定 + promote（grade+3a+fill）
│   ├── llm.py(97)                    LLM 辅助（SKILL_MODEL_* 环境配置）
│   ├── entry_shim.py(92)            确定性 CLI shim（taskspec/--flags 双入口）
│   ├── retrieve.py(77)               catalog→候选（LLM relevance；simple 关键词回退）
│   ├── gate.py(73)                   准入 gate（gold/self_verify/auto）
│   ├── execute.py(65)                run_skill（taskspec→subprocess→agent_response.json）
│   ├── fill.py(64)                   槽位填充（任务文本→params；不猜）
│   ├── library.py(60)                Skill/Library 存储（skill.py+meta.json）
│   ├── prompt.py(48)                 with_skill_hint（库查询提出 agent loop）
│   └── examples/（learned_library 1 技能 + trajectories 3 个 + solve_with_library.sh）
├── exceptions.py                     InterruptAgentFlow(LimitsExceeded/Submitted/FormatError)
├── utils/{logging,runtime,serialize}.py  runtime log / run_async / recursive_merge+UNSET
└── __init__.py                       Model/Environment/Agent Protocol + dotenv/platformdirs
```

### 3.3 入口
- CLI：`webwright`（typer app；`main` 子命令 + `doctor`）——`python -m webwright.run.cli main -t <task> --start-url <url> -o <dir> -c base.yaml -c model_openai.yaml`
- skill_factory：`python -m webwright.skill_factory <init|build|learn|update|route>`（`__main__.py`）
- 工具：`python -m webwright.tools.<skill_use|self_reflection|image_qa|persistent_local_browser>`
- 插件：Claude Code/Codex/OpenClaw/Hermes 共享 `skills/webwright/`（SKILL.md + commands/{craft,run}.md + reference/*）

### 3.4 核心数据结构（代码证据）
- **Observation dict**（local_browser/local_workspace `_capture_observation`）：success/exception/command/returncode/url/title/aria_snapshot/console_output/recent_console/screenshot_path/workspace 文件列表等
- **Action dict**：`{"bash_command"|"python_code": <code>, "command": <同一 code>}`（models/base.py `_query_async` 组装；action_field 二选一）
- **message dict**：`{role, content, extra}`；extra 承载 actions/done/final_response/raw_response/usage/interrupt_type
- **trajectory.json**：格式名 `"webwright-0.1"`（agents/default.py save），messages 数组含 role=exit（extra.exit_status/submission/final_response）
- **skill 目录契约**：`<library>/<skill_id>/{skill.py, meta.json, replays.json?}`（library.py）；meta 含 template/provenance/site/summary/signature{params,call}/output_schema/n_solves/revisions/verified/grade
- **taskspec.json 契约**：`{params, start_url, credentials?, output_schema}` → 技能读 `sys.argv[1]`，写 `agent_response.json`（`{retrieved_data: ...}`）
- **manifest（manual update）**：`{template, runs:[{dir, admit(必须 bool), params, verdict, site?, output_schema?, answer?, credentials?}]}`

### 3.5 状态
- **AgentConfig**（agents/default.py）：step_limit（代码默认 15；base.yaml 配置 100）/keep_last_n_observations/require_self_reflection_success/summary_every_n_steps 等
- **browser 三模式**（local_browser.py）：`local_cdp`（连现有 CDP，探活 /json/version+/json/list，找不到自启 Chrome）`local_launch`（自启）`local_persistent`（持久 context；配合 tools/persistent_local_browser.py 的 .lb_session.json 跨 step 保持浏览器）
- **workspace 两模式**：local_workspace（bash 命令逐条执行，默认环境；base.yaml environment_class=local_workspace, browser_mode=local）/ local_browser（python_code 执行，注入 async def __agent_step__(page,context,browser,playwright,task)）
- **完成门控**：require_self_reflection_success → done 放行需 `final_runs/run_<latest>/self_reflect_result.json` 的 `predicted_label==1`（agents/default.py `_self_reflection_gate_error`）

### 3.6 测试体系（tests/ 1899 行，14 文件）
- `tests/skill_factory/`（12 文件，LLM-free 单元测试，CI 直接 `python file` 跑）：test_build_init(412)/test_evolve(334)/test_route(250)/test_entry_shim/test_execute/test_fill/test_gate/test_learn/test_library/test_llm_env/test_recommend/test_retrieve_decide/test_learned_example(96+)
- `tests/unit/`（2 文件）：test_doctor(94)/test_tool_model_routing(69)
- `tests/conftest.py`(6 行)
- CI：`.github/workflows/skills-tests.yml`——仅 skill_factory/** 与 tools/skill_use.py 路径变更触发；pip install 4 包后 LLM-free 跑全部 test_*.py；wrapper usage check（`solve_with_library.sh` 缺参必须 exit 1）
- 测试关键断言样例（test_evolve.py/test_learned_example.py）：grade 三态互斥、on_fail=reference 永不覆盖已验证技能、增量 refine 必须 regression-replay 旧 coverage、_norm 折叠写法不折叠内容、_memorized_answer 窄判据、manifest 缺 admit loud fail、learned_library 样例必须 n_solves≥3 且 params≥2 且模板无 pipeline 文本泄漏（F7）

### 3.7 配置（config/ 目录 + 栈式合并）
- 文件：base.yaml(23KB)/local_browser.yaml(12KB)/persistent_browser.yaml(30KB)/crafted_cli.yaml(26KB)/task_showcase.yaml(36KB)/model_{openai,claude,openrouter}.yaml
- 合并机制：`-c` 可重复（CLI 默认 `["base.yaml","model_openai.yaml"]"`）→ `get_config_from_spec`（本地路径/builtin/inline `key=value` 点分嵌套）→ `recursive_merge`（UNSET 忽略；dict 递归；后覆盖前）
- snapshot：每次 run 写 `config_snapshot/{manifest, merged_config.yaml, 各 spec 副本}`（可复现；工具模型经 `_model_config.load_tool_model` 读它）
- debug 模式（cli.py）：headless=False/devtools=True/keep_open_on_exit/prompt_before_close/slow_mo 250
- 环境变量：OPENAI_API_KEY/ANTHROPIC_API_KEY（模型）+ OPENAI_ENDPOINT/OPENAI_MODEL/SKILL_MODEL_NAME/SKILL_MODEL_ENDPOINT/SKILL_MODEL_CLASS/SKILL_MODEL_TIMEOUT/SKILL_AGENT_MAX_TOKENS/SKILL_RUN_TIMEOUT/SKILL_LIBRARY_ROOT（skill_factory 侧）+ BROWSERBASE_API_KEY/BROWSERBASE_PROJECT_ID（可选 cloud 浏览器）

### 3.8 权限与治理机制
- SECURITY.md（微软标准模板：安全漏洞不走公开 issue）+ CODE_OF_CONDUCT.md + SUPPORT.md（未编辑模板——仓库早期状态）
- **无 AGENTS.md、无 CONTRIBUTING.md**（ls 实测）
- 治理语义在代码内：credentials 永不写入 skill/replays.json（library 可共享提交）；API key 序列化 redact `<redacted>`；运行时日志剥离敏感内容（`_sanitize_message_for_disk`）；subprocess 沙箱为临时目录+WORKSPACE_DIR 环境约束；local_workspace `_resolve_cwd` 强制 cwd 在 workspace 内

### 3.9 外部依赖/集成
- 浏览器：Playwright（Chromium/Firefox——SKILL.md 插件版用 Firefox 防 TLS/H2 指纹）；CDP 直连（local_cdp）；Browserbase 云会话（browserbase 模式，环境变量条件注入）
- 模型：OpenAI Responses API（json_schema strict）/Anthropic/OpenRouter
- 宿主集成：Claude Code/Codex/OpenClaw/Hermes（`.claude-plugin/plugin.json`、`.codex-plugin/plugin.json`、skills/webwright/）
- 基准：Online-Mind2Web 300 任务 86.7%（GPT-5.4）hard split Opus 4.7 80.5% vs gpt-5.4 76.6%；Odysseys 200 任务 60.1%（avg 76.1 steps，+15.6 over Opus 4.6 44.5%，+26.6 over base 33.5%）；WebArena reuse 55%→70%（README §Benchmarks 原文数字）

## 4. 快照边界与已知未采集
- 已采集：全部 src 代码精读（含 skill_factory 14 模块、tools 4 工具、config、CLI、模型层）+ README 全文 + docs/skill_factory/manual.md + skills/webwright/SKILL.md + base.yaml/instance_template/system_template + tests 关键断言 + CI
- 未细读（阶段④按需）：docs/skill_factory/reference.md、commands/{craft,run}.md、reference/{cli_tool_mode,playwright_patterns,workflow}.md、local_browser.yaml/persistent_browser.yaml/crafted_cli.yaml/task_showcase.yaml 全文、model_{anthropic,openrouter}_model.py 细节
- **无 ADR 目录**（ls 实测）；决策证据来源 = README News/对比表 + docs/manual.md + 代码注释（如 route.py agent_cfg 注释、update.py grade 三态注释）
