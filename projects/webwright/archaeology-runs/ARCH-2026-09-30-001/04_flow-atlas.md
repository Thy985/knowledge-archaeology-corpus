# 04 Flow Atlas（七类流）— microsoft/Webwright

> 关键 Edge 均回溯 symbol/file/condition/state transition。符号约定同 02 层。

## 1. Control Flow（控制流）
```
cli.run_one (cli.py)
  → 校验 task 非空（ValueError）
  → config 栈递归合并（recursive_merge(base, model_*, mode)）
  → get_model / get_environment / get_agent 工厂装配
  → DefaultAgent.run (default.py:341)
      loop:
        step_limit 超限? → LimitsExceeded(exit message)          [default.py:386-392]
        query(model) → strict JSON 解析失败 → FormatError → 注入 format_error_template → continue
        execute_actions (default.py:400):
          done 且 require_self_reflection_success:
            → _self_reflection_gate_error? → done=False, 注入阻塞消息 → continue
          done 且通过门 → Submitted → exit(exit_status=Submitted)
          非 done → env.execute(action) → 观测 → 可选重附模板 → continue
      finally: save(output_path)  每步落盘 trajectory.json
```
关键 Edge：`done=true ∧ predicted_label==1 → Submitted exit`；`done=true ∧ predicted_label≠1 → demote done=False`（default.py:202-268）；`FormatError 计数 n_format_errors`（default.py:383-385）。

## 2. State Flow（状态流）
```
workspace 模式：
  磁盘态 = workspace_dir/{task.json, plan.md, self_reflect_config.json,
        final_script.py, final_runs/run_<id>/{final_script.py, log, screenshots/},
        trajectory.json}
  浏览器态 = 无（每步新会话，代码重建）
  → step 数单调增（n_calls）；summary_every_n_steps=20 → 压缩（状态收敛为 [system, summary]）
live browser 模式：
  浏览器态 = page/context/browser/playwright 跨步持久（lb.yaml）
  消息态 = 观测消息序列，ARIA 只留最近 1（keep_last_n_observations=1, default.py:275）
```
关键 Edge：`_compact_history → 替换全部非 system 消息为单条 summary`（default.py:303）；`_prune_old_observation_aria_snapshots → aria_snapshot 置 "" 并替换占位`（default.py:275-301）。

## 3. Data Flow（数据流）
```
task(-t/--start-url) → instance_template 注入（Jinja2, 含 workspace 元数据）
模型响应(JSON) → parse_json_output(demote done) → env.execute
   workspace: bash_command → subprocess.run(shell=True, /bin/bash, env=command_env) → stdout/stderr
   browser:   python_code → asyncio.wait_for(run_python_code, timeout) → output/returncode/exception
观测 → observation_template 渲染（command_output/aria_snapshot/screenshot_path…）→ 消息回灌
收尾数据：
  final_script.py 运行 → final_script_log.txt + screenshots/*.png
  self_reflection → self_reflect_result.json {per-image records, final prompt, predicted_label}
trajectory.json 每步 save（output_path）
技能沉淀：learn → library/<skill_id>/{skill.py, meta.json, replays.json} + .learned.json ledger
```
关键 Edge：`_task_metadata_path = workspace/task.json`（lw.py:94）；`OM2W_TASK_JSON/FINAL_SCRIPT_PATH env 注入`（lw.py:185-193）。

## 4. Evidence Flow（证据流）
```
agent 探索期 → 截图 screenshots/*.png（image_qa 主动调用）
agent 收尾 → final_script 从零跑 → critical-point 截图 + action log
self_reflection（两阶段）:
  Stage1: 每图 (system, user+image) → "Reasoning:\nScore: 1-5" → retry ≤3
  Stage2: {image_reasonings} + {action_history_log} → 聚合 → "Status: success|failure"
  → predicted_label(1/0/null) → self_reflect_result.json
外部 judge 读同一文件（证据链闭环）
```
关键 Edge：`final verdict 缺失/畸形 Status → exit 1（FAIL）`（self_reflection.py:283）；`ParseFailed → Score:0 不弃 run`（容错）；`transient HTTP → 指数退避`。

## 5. Authority Flow（权威流）
```
完成权：Self-Reflection Gate（require_self_reflection_success=true）
  done=true → gate 检查 → predicted_label==1 → Submitted exit
  否则 → 阻塞消息（含具体修复指引）→ agent 继续修
技能准入权：skill_factory gate
  gold 存在 → 精确匹配（method=gold/auto）
  无 gold → self_verify（非空 + shape + status）——弱门（learn.py 自证警告）
执行权：模型无直接系统权限；全部经 env.execute（bash/python 子进程）
  凭证权：OPENAI_API_KEY 等 env 校验（缺失 RuntimeError）
```
关键 Edge：`_tool_gate_error`（default.py:202）是完成权的硬拦截点；`gate.admit=False → 不进分组`（learn.py:223）。

## 6. Memory Flow（记忆流）
```
运行内记忆：
  - 消息序列（上下文）→ summary_every_n_steps=20 LLM 压缩（_compact_history）
  - ARIA 快照裁剪（keep_last_n_observations=1）
  - explore_history 复用（先前探索日志注入新 run, default.py:352-365）
跨运行记忆（程序化，loop 外）：
  - library/<skill_id>（skill.py 可无模型重跑）
  - replays.json（重放证据）
  - .learned.json（已学 ledger, learn.py）
  - examples/learned_library（自带示例技能）
```
关键 Edge：`route → run 时零模型执行；adapt 时 skill 注入 prompt 作 prior`（route.py）。

## 7. Policy Flow（策略流）
```
决策 → 策略 → 固化 → 执行 → 影响后续决策
  Task Success Criteria 8 条（filter 必须用控件/数值精确/ranking 落真实指标…）
    → base.yaml system_template/instance_template 固化
    → agent 执行动作受其约束
    → Completion Gate 5 条把策略变成 done 前置条件
  require_self_reflection_success / keep_last_n_observations / summary_every_n_steps
    → config 固化 → env/agent 运行时执行
  gate 策略（gold > self_verify）/ route 策略（run/adapt/skip + fallback）
    → skill_factory 代码固化 → 每次 learn/route 生效
  agent_cfg 策略（-c 重加 base.yaml；max_output_tokens 16000）
    → route.py 注释固化（防网关 401 / 防截断循环）
```
关键 Edge：`policy 的 enforcement 点 = gate/_self_reflection_gate_error/config 加载`；策略变更路径 = 改 yaml/改 prompt（无运行时策略引擎）。

## Flow→KO 交叉校验
- KO-01 完整对应 Authority Flow（完成权授权链）。
- KO-02 对应 Memory Flow 运行内治理 + Control Flow 压缩点。
- KO-04 对应 Data Flow 技能沉淀 + Memory Flow 跨运行 + Authority Flow 准入。
- KO-08 对应 Authority Flow 的模式依赖分支（workspace 门 vs browser 自判）。
- 无 KO 与 Flow 矛盾。
