# 01 Project Layer — microsoft/Webwright

## 1.1 项目定位与历史
- 微软研究院开源终端原生 Web Agent 框架（2026-05-04 发布，2026-08-03 HEAD bc26750a）。
- 论文：《Webwright: A terminal is all you need for web agents》(Lu, Xu, Huang, Awadallah, 2026)。
- 演进：初始 ~1.5k LoC → 插件化（Claude Code/Codex/OpenClaw/Hermes）→ Task2UI → **Skill Factory**（2026-07-21）→ 模块化重构（#62）。

## 1.2 架构总览
```
CLI (run/cli.py, typer) ── config 栈递归合并（base + model_* + mode）
  └─ get_agent / get_environment / get_model（工厂）
      └─ DefaultAgent.run（agents/default.py: 467 行）
          ├─ query() → BaseModel.query（strict JSON + 重试退避）
          ├─ execute_actions() → env.execute(action)
          │    ├─ LocalWorkspaceEnvironment（bash, 240s 超时, 磁盘状态）
          │    └─ LocalBrowserEnvironment（python_code, CDP/持久/本地, 浏览器状态）
          └─ 观测 → format_observation_messages → 循环
  └─ tools: image_qa / self_reflection / skill_use / persistent_local_browser
  └─ skill_factory: init / build / learn / update / route / gate / library…
宿主插件：.claude-plugin / .codex-plugin / skills/webwright（四宿主共用）
```

## 1.3 核心抽象
| 抽象 | 载体 | 职责 |
|---|---|---|
| Model（后端无关协议） | models/base.py `BaseModel` | strict JSON 解析、done 降级、bash -n 预检、rate-limit/transient 重试、usage 指标、key 校验 |
| Environment（状态宿主） | environments/* | 动作执行 + 观测捕获；workspace（磁盘状态）/ browser（浏览器状态）两实现 |
| Agent（loop） | agents/default.py `DefaultAgent` | 模板注入、步进、完成门、上下文治理（ARIA 裁剪/LLM 压缩）、debug 工件 |
| Skill（程序化记忆） | skill_factory/library.py `Skill` | skill.py + meta.json + replays.json；可无模型重跑 |
| Gate（准入/完成判定） | skill_factory/gate.py + tools/self_reflection.py | 技能准入（gold/self_verify）；完成授权（predicted_label==1） |

## 1.4 生命周期（一次 solve）
1. CLI 组装 config → 生成时间戳输出目录 + 快照 config（snapshot_config_specs）
2. Agent.run：注入 system/instance 模板（含 Completion Gate 清单）→ 循环 step
3. 每步：模型出 JSON（thought + 动作 + done）→ env.execute → 观测 → 回灌
4. 探索期：写脚本探站点、存 plan.md（critical points）、author self_reflect_config.json
5. 收尾：写 final_script.py → 在 final_runs/run_<id>/ 从零执行 → 跑 self_reflection（exit 0 + predicted_label==1）→ 才允许 done=true
6. skill_factory learn：gate 准入 → 分组 → 蒸馏参数化 skill 入库
7. 未来任务：route 检索库 → run（零模型）/ adapt（hint）/ skip

## 1.5 数据流
- 输入：task + start_url + config yaml 栈
- 中间：trajectory.json（每步）、screenshots/*.png、final_runs/run_<id>/{final_script.py, final_script_log.txt, screenshots/}
- 判定：self_reflect_result.json（predicted_label）
- 沉淀：library/<skill_id>/{skill.py, meta.json, replays.json}；.learned.json ledger

## 1.6 权限与治理
- 无内部权限模型；凭据：OPENAI_API_KEY / ANTHROPIC_API_KEY / BROWSERBASE_API_KEY+PROJECT_ID（缺失 → RuntimeError；serialize 脱敏 `<redacted>`）。
- 治理 = Completion Gate 五条 + Task Success Criteria 八条（prompt 级策略，非代码级 enforcement）。

## 1.7 测试与 CI
- 14 文件 1899 行 LLM-free 单测；CI 按路径过滤触发（skill_factory/** 变更才跑）；F6 wrapper usage check。

## 1.8 配置矩阵（9 yaml）
base（workspace 模式）/ model_openai / model_claude / model_openrouter / local_browser / persistent_browser / task_showcase / crafted_cli；`-c` 会替换默认（agent_cfg 强制重加 base.yaml）；`DEFAULT_CONFIGS = ["base.yaml", "model_openai.yaml"]`。

## 1.9 边界与例外
- workspace 模式禁 pip/apt 安装；禁 full_page 截图（viewport 1280x1800 强制）；禁 done 与非空动作同响应（代码 + prompt 双强制）。
- live browser 模式禁写文件、禁 image_qa/self_reflection、禁自关浏览器。
