# 01 Project Layer — Webwright 项目地图

## 1. 是什么 / 怎么运行
Webwright = LLM 网页 agent harness（SWE-style）。一次 run：CLI 装配 config → 实例化 Model/Environment/Agent → `env.prepare(task, task_id, start_url)` → `agent.run(task)`（step loop：模型输出 strict JSON → 执行 1 条 bash/python 命令 → observation 回流 → 直到 done gate 通过）→ `env.close()`。产物：`<output_dir>/<task_id>_<timestamp>/`（trajectory.json + script.py + steps/ + screenshots/ + logs/ + config_snapshot/）。

## 2. 架构（三层）
```
CLI (run/cli.py, typer) + doctor
  └─ run_one: DEFAULT_CONFIGS=[base.yaml, model_openai.yaml] → recursive_merge → config_snapshot
        ├─ Model   (models/: base.py 底座 + openai/anthropic/openrouter 子类；get_model 工厂)
        ├─ Environment (environments/: local_browser | local_workspace；get_environment 工厂)
        └─ Agent   (agents/default.py；get_agent 工厂)
              └─ step loop: query(model) → execute(env) → observation → _self_reflection_gate → done?
skill_factory（独立管线）：init → build → learn → update/route → library
  └─ examples/: learned_library + trajectories + solve_with_library.sh
tools/：self_reflection(两阶段裁判) / image_qa / skill_use(recommend) / persistent_local_browser / _model_config
插件层：skills/webwright/（Claude Code/Codex/OpenClaw/Hermes 共享）
```

## 3. 核心模块与职责
| 模块 | 职责 | 关键机制 |
|---|---|---|
| agents/default.py (467) | agent loop、completion gate、上下文压缩 | step_limit、require_self_reflection_success、InterruptAgentFlow、_sanitize_message_for_disk |
| environments/local_browser.py (567) | 浏览器执行环境（三模式） | local_cdp/local_launch/local_persistent；CDP 探活+自启；`async def __agent_step__(page,context,browser,playwright,task)` 注入 |
| environments/local_workspace.py (296) | bash 工作区环境 | 逐条 subprocess 执行；cwd 沙箱；steps/ + command_history.sh + 截图/日志持久化；BROWSER_MODE 透传 |
| models/base.py (587) | provider-agnostic 底座 | strict JSON schema；retry 分类；bash 语法预检；usage 双轨指标；redact |
| run/cli.py (175) | 装配与执行 | config 栈式合并；时间戳目录；debug 模式；异常分离 |
| skill_factory/* (2150) | 技能蒸馏流水线 | gate→group→canonicalize→evolve(refine/replay)→grade；route run/adapt/skip；recommend 护栏 |
| tools/self_reflection.py (611) | 两阶段截图裁判 | Stage1 逐图 Score+Reasoning；Stage2 聚合 Status: success/failure |
| config/*.yaml | 策略配置 | base.yaml(agent/system_template/instance_template)/local_browser/persistent_browser/crafted_cli/task_showcase + model_*.yaml |

## 4. 生命周期（一次 run）
1. CLI 解析 `-t/--start-url/--task-id/-o/-c` → 合并 config → `snapshot_config_specs`
2. `env.prepare`：建 workspace 目录（outputs/…/.tmp + steps/logs/screenshots）+ 写 task.json
3. `agent.run`：`_step_loop` 迭代至 step_limit——
   - 组装 messages（system_template + instance_template + observation 历史 + 可选 skill hint）
   - `model(messages)` strict JSON → `parse_json_output`（修复循环）
   - `_validate_bash_command`（/bin/bash -n）→ `env.execute(action)`
   - observation 回流；`summary_every_n_steps` 压缩；done gate（self_reflect_result.json predicted_label==1）
   - `InterruptAgentFlow`（LimitsExceeded/Submitted/FormatError）→ exit
4. `env.close`（finally）→ trajectory.json（role=exit 含 exit_status）

## 5. 生命周期（Skill Factory）
`init`（需求→skill.yaml 草稿，drift 预判）→ `build`（spec→N 实例并行求解+learn）→ `learn`（run 目录→gate→分组→canonicalize→evolve）→ `update`（manual manifest→evolve）→ `route`（新任务：retrieve→decide→promote→run/adapt/skip，直接跑失败回退 agent）

## 6. 关键配置（base.yaml 实测）
- model: request_timeout 120 / max_output_tokens 4000 / attach_observation_screenshot false / observation_template / format_error_template
- environment: environment_class local_workspace / browser_mode local / command_timeout 240 / shell /bin/bash / output_truncation_chars 24000
- agent: step_limit 100（代码默认 15 被覆盖）/ require_self_reflection_success true / summary_every_n_steps 20 / debug_log true
- system_template：strict JSON 单对象；bash_command 单命令；done 与 action 不同 turn；禁 full_page screenshot；view 1280x1800；禁 pip/apt 安装
- instance_template：plan.md 关键点清单；final_runs/run_<id>/ 结构；final_script.py 仪表化；self_reflect_config.json 四提示词；completion gate 5 条

## 7. 测试体系（可执行知识）
- LLM-free 单元测试 14 文件（skill_factory 12 + unit 2），CI 直接 `python file` 跑
- 关键守护断言：grade 三态互斥（executable/reference/unverified）；on_fail=reference 永不覆盖已验证技能；增量 refine regression-replay 旧 coverage；_norm 折叠写法不折叠内容（AS26≠AS27）；manifest 缺 admit/非 bool loud fail；learned_library 样例 n_solves≥3、params≥2、无 pipeline 文本泄漏（F7）、artifact 不锚定 __file__；wrapper 缺参 exit 1
- CI 路径门：仅 skill_factory/** + tools/skill_use.py 变更触发

## 8. 权限与治理
- SECURITY.md/CODE_OF_CONDUCT.md（微软模板）；无 AGENTS.md/CONTRIBUTING.md
- 代码内治理：credentials 不落 skill/replays.json；API key redact；_sanitize_message_for_disk；cwd 沙箱；subprocess 超时；gate/verify 策略显式化
- 插件清单：.claude-plugin/plugin.json + .codex-plugin/plugin.json

## 9. 外部依赖
Playwright（Chromium/Firefox）/ CDP / Browserbase（可选 cloud）/ OpenAI·Anthropic·OpenRouter / httpx·pydantic·jinja2·pyyaml·rich·typer·dotenv·platformdirs
