# 02 Engineering Knowledge（EK Graph）— microsoft/Webwright

> 每 EK 声明 `links`（六类边：mechanism/subsystem/causal/dependency/constraint/contrast）；KO 从图上按 R1-R4 聚合。证据 = 文件路径 + 符号，全部可回溯。

## 证据符号约定
- `default.py:341` = src/webwright/agents/default.py 行 341
- `lb.py:387` = environments/local_browser.py；`lw.py:176` = environments/local_workspace.py
- `base.yaml:<键>` = config/base.yaml；`lb.yaml:<键>` = config/local_browser.yaml
- `mb.py` = models/base.py；`gate.py`/`route.py`/`learn.py`/`update.py`/`library.py`/`retrieve.py`/`fill.py` = skill_factory/*.py

## EK 列表（44 条）

### 核心机制（M）
| ID | 层 | 内容 | 证据 | links |
|---|---|---|---|---|
| EK-01 | L2 | **Code-as-action**：动作面 = 自由形式 Python/bash 脚本（agent 自己写 Playwright 代码），非离散动作空间；每步单 JSON `{thought, action_field, done, final_response}` | default.py:341-437; base.yaml system_template; mb.py _response_schema | mechanism→EK-02, EK-03; contrast→EK-04(browser-use 对照, corpus) |
| EK-02 | L2 | **workspace-as-state**：浏览器无持久状态，每步新开会话、用代码重建；状态在磁盘（workspace_dir/final_runs/trajectory.json） | base.yaml system_template "NO persistent browser state"; lw.py _workspace_dir | mechanism→EK-01; subsystem→EK-05; contrast→EK-06(live browser) |
| EK-03 | L2 | **执行-观测-修复循环**：write code → execute（超时 240s）→ capture observation（截图+aria+console）→ 回灌模型 → 修复脚本 | default.py:400-437; lw.py:176-220 | causal→EK-07, EK-09; mechanism→EK-01 |
| EK-04 | L2 | **双模式一个 loop**：workspace（无状态、bash、磁盘产物、外部验证）与 live browser（有状态、python_code、ARIA、模型自判）共用 DefaultAgent；差异收敛于 config/action_field | lb.yaml 全部; base.yaml environment.environment_class | contrast→EK-02; subsystem→EK-01; causal→EK-12 |
| EK-05 | L2 | **观测模板降载**：screenshots 默认不自动 attach（attach_observation_screenshot: false）；以 ARIA 快照 + 打印文本为观测主通道；需要时显式调 image_qa | base.yaml model.attach_observation_screenshot; base.yaml system_template "Step screenshots are NOT automatically attached" | dependency→EK-13; constraint→EK-14 |
| EK-06 | L2 | **live browser 状态持久**：page/context/browser/playwright 跨步暴露给 agent；禁自 launch/close；browser_mode=local_cdp 接真实 Chrome/Edge | lb.yaml system_template "The harness already exposes…"; lb.py _ensure_local_cdp_browser | contrast→EK-02; subsystem→EK-04 |
| EK-07 | L2 | **strict JSON 协议**：required 四字段、additionalProperties: false、禁 prose/code fences；解析失败 → FormatError → 重加格式错误消息 | mb.py _response_schema; default.py:383-385; base.yaml format_error_template | causal→EK-09; constraint→EK-08 |
| EK-08 | L2 | **done 降级防御**：action 非空 + done=true → parse 时自动 demote done=false（防"同一响应既执行又宣称完成"） | mb.py:107-121 | constraint→EK-07; mechanism→EK-10 |
| EK-09 | L1 | **bash -n 语法预检**：执行前 subprocess `/bin/bash -n` 校验命令语法 | mb.py _validate_bash_command | causal→EK-03; dependency→EK-08 |
| EK-10 | L2 | **中断流协议**：InterruptAgentFlow 携带消息注入；FormatError 计数 n_format_errors；LimitsExceeded 触发 exit | exceptions.py; default.py:341-380 | mechanism→EK-08; causal→EK-11 |
| EK-11 | L2 | **step_limit 硬上限**（100）：query 前检查，超限 → LimitsExceeded exit 消息 | default.py:386-392; base.yaml agent.step_limit | causal→EK-10; constraint→EK-01 |
| EK-12 | L2 | **Self-Reflection Gate（完成授权门）**：require_self_reflection_success=true 时 done 被阻塞，直到 final_runs/run_<latest>/self_reflect_result.json 的 predicted_label==1；否则返回详细阻塞原因消息 | default.py:202-268; base.yaml agent.require_self_reflection_success | causal→EK-15, EK-16; constraint→EK-07 |
| EK-13 | L2 | **ARIA 快照裁剪**：keep_last_n_observations=1（browser 模式），旧 observation 的 aria_snapshot 替换为占位符"pruned"（ARIA 10-20k chars/个，主导 token 消耗） | default.py:275-301; lb.yaml model.keep_last_n_observations | dependency→EK-05; mechanism→EK-14 |
| EK-14 | L2 | **LLM 历史压缩**：summary_every_n_steps=20 → _compact_history 用 LLM 把 [全部消息] 压成 [system, summary]；压缩失败不阻断 run | default.py:303-340; base.yaml agent.summary_every_n_steps | mechanism→EK-13; constraint→EK-05 |
| EK-15 | L2 | **两阶段图片判定器**：Stage1 per-image Score 1-5 + Reasoning（retry 3 次，失败记 Score:0 ParseFailed:true）；Stage2 注入全部 reasoning + action log → 单次聚合 → `Status: success|failure` 结尾行 | self_reflection.py:186-283; base.yaml system_template Task Reflection Tool | causal→EK-12; mechanism→EK-16 |
| EK-16 | L2 | **判定产物即授权凭证**：self_reflect_result.json（含 predicted_label 1/0/null）是唯一完成授权来源；外部 judge 读同一文件 | default.py _tool_gate_error; base.yaml Completion Gate #4 | causal→EK-12; constraint→EK-15 |
| EK-17 | L2 | **Completion Gate 五条硬清单**：plan.md + self_reflect_config.json + final_script 从零执行 + self_reflection exit 0 + 工件 ls/cat 确认 | base.yaml Completion Gate; instance_template Step 6 | constraint→EK-12; causal→EK-18 |
| EK-18 | L2 | **final_script 从零重跑**：每次干净执行放 final_runs/run_<id>/（整数递增 id），含 final_script_log.txt + critical-point screenshots；id+1 迭代修复 | base.yaml instance_template Harness Rules | causal→EK-17; dependency→EK-12 |
| EK-19 | L2 | **Task Success Criteria 策略**：专用控件必须用（filter/sort 不靠搜索词）、数值精确匹配不宽化、ranking 用语须落站点真实指标、隐藏控件先展开再判不存在 | base.yaml instance_template Task Success Criteria; system_template Rules | constraint→EK-17; subsystem→EK-20 |
| EK-20 | L2 | **证据密度策略**：critical points 每项独立可验证（截图或日志），多存关键点截图提高 judge 通过率；final_response 显式给出最终数据 | base.yaml system_template Rules "The more evidence you save…"; instance_template Final Script Instrumentation | subsystem→EK-19; causal→EK-15 |
| EK-21 | L2 | **模型后端三态**：OpenAI/Anthropic/OpenRouter 各自 ~150-200 行子类；BaseModel 协议（headers/url/payload/extract/usage 六扩展点）+ 默认 retry（rate-limit 5 次退避 min(5*(a+1),30)、transient 5 次退避 min(2*(a+1),10)） | models/{base,openai,anthropic,openrouter}_model.py; mb.py _rate_limit_backoff | subsystem→EK-22; mechanism→EK-10 |
| EK-22 | L2 | **API key 缺失即失败**：构造时 env 校验，缺失 → RuntimeError；serialize 时 key 脱敏 `<redacted>` | mb.py __init__; serialize | subsystem→EK-21; constraint→EK-23 |
| EK-23 | L2 | **usage 指标透传**：last/cumulative request+usage 指标注入模板变量（get_template_vars）供观测提示词引用 | mb.py get_template_vars; _usage_snapshot | dependency→EK-05; subsystem→EK-21 |

### 关键实现/决策（I）
| ID | 层 | 内容 | 证据 | links |
|---|---|---|---|---|
| EK-24 | L2 | **config 栈递归合并**：-c 可叠加；`-c` 会替换默认 → agent_cfg 强制重加 base.yaml（否则 agent 无法构造）；支持 OPENAI_ENDPOINT/MODEL 网关内联构造 config | run/cli.py run_one; route.py agent_cfg 注释 | constraint→EK-25; causal→EK-01 |
| EK-25 | L2 | **max_output_tokens 陷阱**：base.yaml 4000 会在复用大 skill 时截断 agent 中途（run 循环）；agent_cfg 提到 16000（SKILL_AGENT_MAX_TOKENS 可覆盖） | route.py agent_cfg 注释 | constraint→EK-24; causal→EK-01 |
| EK-26 | L2 | **时间戳输出目录**：outputs/<task_id>_<YYYYMMDD_HHMMSS>；config 快照 snapshot_config_specs 随 run 落盘（可复现） | run/cli.py _timestamped_output_dir; run_one | subsystem→EK-24; mechanism→EK-18 |
| EK-27 | L2 | **explore_history 复用**：把先前 live-browser 探索的消息日志注入新 run 作为 prior（"Do NOT repeat failed approaches"） | default.py:352-365 | mechanism→EK-33; contrast→EK-05 |
| EK-28 | L2 | **观测后重附模板**：attach_instance_template_after_observation / attach_plan_md_after_observation 可在观测后重注入任务/计划上下文（防目标漂移） | default.py:424-436 | constraint→EK-14; mechanism→EK-27 |
| EK-29 | L2 | **debug 工件逐步入盘**：每步写 step artifact（assistant message + outputs）到 debug 目录，失败也可审计 | default.py _write_debug_step_artifact | causal→EK-03; subsystem→EK-26 |
| EK-30 | L2 | **doctor 环境诊断**：python 版本/playwright/chromium 等逐项检查给出 Fix 提示 | run/doctor.py check_python/check_playwright/check_chromium | subsystem→EK-24; mechanism→EK-26 |

### Skill Factory（S）
| ID | 层 | 内容 | 证据 | links |
|---|---|---|---|---|
| EK-31 | L2 | **技能=程序非上下文**："Most agent skills are context the model reads. Ours are programs."——skill = skill.py（含 entry_shim CLI）+ meta.json + replays.json | library.py Skill/Library; README Skill Factory | mechanism→EK-32, EK-40; causal→EK-38 |
| EK-32 | L2 | **入口 shim 双模式确定性**：auto-injected CLI 支持 `python skill.py taskspec.json`（replay 路径，sys.argv 原样）与 `--flag` 人性化路径，两路结果 IDENTICAL；缺参 → usage + exit 1（F6 校验） | entry_shim.py; skills-tests.yml F6 check | mechanism→EK-31; constraint→EK-37 |
| EK-33 | L2 | **复用三决策**：route = run（skill 可执行+覆盖+槽全填 → 直接跑，无 agent；失败 crash/timeout/空/错形状 → 带 fallback 标记交 agent）/ adapt（skill 作 prior 交 agent）/ skip（无 hint 从零） | route.py run_webwright; test_route.py:20-78 | causal→EK-38, EK-35; mechanism→EK-27 |
| EK-34 | L2 | **缺槽不猜测 + 决策分层**：fill 槽填充失败/缺参 → adapt 而非硬填（promote_missing_slot_is_adapt_not_a_guess）；decide 只判 shape-fit，promote 才判 run/adapt（grade + 3a + fillable slots）；recommend 三重防御——库缺失显式 warning（防相对路径误判）、**decided skill_id 必须 ∈ retrieve 候选集（LLM 幻觉 id 强制 skip）**、空库不静默 skip | fill.py fill_params; test_recommend.py:38; tools/skill_use.py recommend（Auditor 补） | constraint→EK-33; dependency→EK-36 |
| EK-35 | L2 | **fallback 标记语义**：run 失败回退标记 `tried and failed` + skill 名 hint；adapt 仅 `ADAPT: reuse the core`（不宣称已试） | test_route.py:29-54 | mechanism→EK-33; causal→EK-38 |
| EK-36 | L2 | **相关性检索双法**：llm（LLM 检索 k=3）与 simple（启发式）；结果 Candidate{skill_id, template, source} | retrieve.py retrieve/_retrieve_llm/_retrieve_simple | dependency→EK-34; subsystem→EK-31 |
| EK-37 | L2 | **技能准入 gate**：gold（精确匹配）/ self_verify（非空+形状+status）/ auto（gold 优先，无 gold 走 self_verify）；shape 检查 type=array/object/string/number | gate.py gate/_shape_ok/_self_verify/_gold; test_gate.py 全 | constraint→EK-32; causal→EK-39 |
| EK-38 | L2 | **learn 蒸馏管道**：collect_runs → gate 先筛（错误 solve 永不进分组）→ chunk=25 分组 → 蒸馏为参数化 skill；self_verify 无 gold 时诚实警告"agent 误信的答案仍 PASS" | learn.py learn; test_learn.py | causal→EK-33, EK-39; dependency→EK-37 |
| EK-39 | L2 | **update/evolve 增量生长**：USE（成功不动）/ ADAPT（fix 回蒸馏 widen/harden）/ SKIP（无 skill 则新增）；`_memorized_answer` 检测——脚本含完整答案 verbatim → 丢弃（无方法可蒸馏）；regression-replay 防旧覆盖破坏 | update.py evolve/_memorized_answer; test_evolve.py | causal→EK-38; dependency→EK-37 |
| EK-40 | L2 | **零 token 重放**：learned skill 无模型独立跑 ~40s；replays.json 存重放证据；WebArena reuse 55%→70%（README 声明，Observation 级）。**诚实边界（Auditor 补）**：`_well_shaped` 声明 direct-run 答案只有 shape 保证（无 ground truth），非正确性检查 | library.py replays.json; examples/learned_library/*/skill.py; route.py _well_shaped | mechanism→EK-31; causal→EK-41 |
| EK-41 | L2 | **recommend 注入 prior**：solve 前库检索在 loop 外（`{verdict: run|adapt|skip, skill_id, source_path}`）；agent 复用 hint 而不自查询库 | prompt.py prepend SKILL-LIBRARY hint; README Skill Factory Reuse | causal→EK-40; subsystem→EK-36 |

### 失败与修复 / 边界与例外（F）
| ID | 层 | 内容 | 证据 | links |
|---|---|---|---|---|
| EK-42 | L2 | **格式错误自愈**：模型输出非 JSON → ValueError → FormatError 注入 format_error_template → agent 重出 | mb.py parse_json_output; default.py:383-385; base.yaml format_error_template | mechanism→EK-10; contrast→EK-07 |
| EK-43 | L2 | **超时与重试边界**：命令 240s 超时（workspace）/ step_execution_timeout_ms（browser）→ returncode=-1 + exception_info；rate-limit/transient 指数退避封顶（30s/10s） | lw.py:176-220; lb.py _execute_async; mb.py backoffs | causal→EK-03; subsystem→EK-21 |
| EK-44 | L2 | **模式安全例外**：workspace 禁 pip/apt/full_page 截图（viewport 1280x1800 强制）；live browser 禁写文件/禁 image_qa/self_reflection/禁自关浏览器；blocker 声明须重复 UI 证据 | base.yaml + lb.yaml Rules; system_template | constraint→EK-02, EK-06; subsystem→EK-19 |

## EK Graph 质量
- 44 条 EK，46 条边；平均出边 1.05；游离 EK 0（每条 ≥1 links）；聚合规则覆盖率 100%（见 03）。
