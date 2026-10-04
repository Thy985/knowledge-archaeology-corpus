# 04 Flow Atlas — Webwright 七类流

> 全部从真实代码导出；关键 Edge 标注 `symbol / file / condition / state transition`。

## 1. Control Flow（harness run）
```
webwright (run/cli.py#app)
  → run_one (cli.py)
      → get_config_from_spec(-c 栈) → snapshot_config_specs(config_snapshot/)
      → get_environment(environment_class) → env.prepare(task, task_id, start_url)   # 建 outputs/<taskid>_<ts>/ + task.json
      → get_agent(agent_class, cfg) → agent.run(task)
          → _step_loop (agents/default.py)
              → _query_async (models/base.py)          # strict JSON；retry(9)；parse 修复(8)；bash 预检(6)
              → action = parse_json_output
              → env.execute(action)                     # local_workspace.execute | local_browser._execute_async
              → observation 回流 (2) → messages
              → 每 20 步 _compact_history (18)
              → done? → require_self_reflection_success → _tool_gate_error (17)  ← self_reflect_result.json predicted_label==1
      → env.close() (finally) → trajectory.json (role=exit, exit_status)
Edge: _step_loop 内 done 声明必须"上一 turn 已执行并验证 final script"（base.yaml system_template：NEVER set done in same response as non-empty bash_command）
```

## 2. State Flow
```
状态载体：AgentConfig(messages, step_index, model_cfg) + workspace 文件系统 + 浏览器进程/会话
  AgentConfig.step_limit: 代码默认 15 ← 配置 base.yaml agent.step_limit=100（配置覆盖代码默认）
  messages 组装：system_template + instance_template(任务/URL/workspace/final_script_path) + observation 历史(+可选 skill hint)
  browser state：local_cdp→探活失败才自启；local_persistent→context 持久化跨 step；base 语义=无持久浏览器状态（每次从头重建）
  workspace state：steps/step_NNNN.{py,sh} + screenshots/step_NNNN.png + logs/step_NNNN.log + command_history.sh + script.py + final_runs/run_<id>/*
  skill_factory state：library 目录（skill.py+meta.json+replays.json）；ledger .learned.json（幂等，rejected 不入账）
Edge: cwd 强制 workspace 内（local_workspace._resolve_cwd ValueError 拦截）
```

## 3. Data Flow
```
task → instance_template(task, task_id, start_url, workspace_dir, task_metadata_path, final_script_path)
  → messages[{role, content, extra}]（extra: actions/done/final_response/raw_response/usage）
  → OpenAI: _serialize_response_input（system→developer；assistant→output_text；exit 跳过）
  → payload {model, input, max_output_tokens, text:{format:{type:"json_schema", strict:true, schema:_response_schema()}}}
  → response → _extract_response_text（output_text 优先，回退遍历 output[].content）
  → parse_json_output（done+action 降级）→ env.execute
  → observation（url/title/aria_snapshot/screenshot/console/…，截断 12000/24000）
skill_factory：task 文本 → fill_params（槽位）→ taskspec.json{params,start_url,credentials?,output_schema}
  → subprocess: python skill.py taskspec.json → agent_response.json{retrieved_data}
  → _recover_answer（①agent_response.json ②trajectory exit message ③skip+reason）
```

## 4. Evidence Flow
```
plan.md（关键点清单，每个 CP 可独立验证）
  → final_script.py 仪表化：final_runs/run_<id>/final_script_log.txt（step <n> action 行 + 最终 datum）
  → screenshots/final_execution_<step>_<action>.png（每 CP 至少一张）
  → self_reflection Stage1：逐图 Score 1-5 + Reasoning（retry≤3，ParseFailed 记 0 不炸批次）
  → Stage2：{image_reasonings} + {action_history_log} 聚合 → 末尾 Status: success/failure（最后匹配；缺失=exit 1）
  → self_reflect_result.json{predicted_label:1|0|null}
  → done gate 读取（agents/default._tool_gate_error）← 外部 judge 亦读同一文件
Edge: 模型提示词要求"尽量多存关键点证据（越多越容易 judge 通过）"——证据充分性由 agent 自控，判决由外部裁判
```

## 5. Authority Flow
```
subprocess 执行权：bash 语法预检(/bin/bash -n) → FormatError；timeout 240s；cwd 沙箱；禁 pip/apt 安装
  （base.yaml Rules：Do NOT install additional packages）
模型输出权：strict JSON schema（additionalProperties:false）+ 单命令约束
技能执行权：grade==executable + template_gap 3a + 槽位全填充 → promote 才 run（直接执行权分级授予）
凭证权：credentials 仅 manual-mode replay 携带；永不写入 skill/replays.json；API key 序列化 redact(<redacted>)；_sanitize_message_for_disk
文件权：只写 workspace_dir；artifact 路径禁止锚定 __file__（test_learned_example 断言）
Edge: decide 的 skill_id 必须 ∈ retrieved 候选（LLM 幻觉 id 被拒绝——工具层对模型输出的信任边界）
```

## 6. Memory Flow
```
进程内：messages 历史 + keep_last_n_observations + _compact_history（每 20 步，失败不终止）
磁盘：trajectory.json（webwright-0.1 格式，含 role=exit）+ steps/ + logs/ + config_snapshot/
技能库：library 目录（每 skill_id：skill.py + meta.json{template,provenance,n_solves,revisions,verified,grade} + replays.json）
  → update 增量 refine 把旧 replays 并入回归集（旧 coverage 不被破坏）
  → .learned.json ledger 幂等（rejected run 可重学）
浏览器记忆：默认无（每次从头导航）；persistent_local_browser 会话 JSON{id,pid,connectUrl,userDataDir} 显式记忆
Edge: _compact_history 压缩失败 → run 继续（压缩是优化非正确性依赖——记忆管理不阻断主流程）
```

## 7. Policy Flow
```
治理闭环 Decision→Approval→Policy→Enforcement→Future Decision
  Decision：README 范式声明（code-as-action、no hidden orchestration）→ News 2026-07-21 Skill Factory 决策
  Approval：PR #62（Add Web Skill Factory）进 main；CI skills-tests.yml 路径门（仅 skill_factory/**+skill_use.py 变更触发）
  Policy 固化：
    - base.yaml system_template 硬规则（单命令/禁安装/禁 full_page/1280x1800/ranking 必须 grounded/数值约束 exact）
    - instance_template Task Success Criteria 8 条（filter 必须用站点控件；broad search 不满足 filter）
    - Completion Gate 5 条（plan.md + self_reflect_config.json + run 产物 + predicted_label==1 + ls 确认）
    - skill_factory gate 策略（gold/self_verify/auto）+ verify: strict|shape（drift 判定）+ grade 三态
  Enforcement：_self_reflection_gate_error 打断 done；gate.py 过滤；_memorized_answer 弃料；fill 不猜 null；route fallback
  Future Decision：WebJudge（OM2W 官方 judge）或跨源一致性作为 self_verify 升级路径（gate.py 注释）
Edge: 策略文本（system/instance template）以 jinja2 渲染进 prompt 的"外部化策略"，与代码内 gate 双轨执行
```
