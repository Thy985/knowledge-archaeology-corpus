# 02 Engineering Knowledge — Webwright（48 EK，EK Graph）

> 宽底座 · 推理原材料。每条 EK 声明 `links`（六类边：mechanism/subsystem/causal/dependency/constraint/contrast）。
> 证据引用格式：`file.py#symbol|行区`（符号/行号可回溯）。EK 不因形成 KO 而删除；KO 可反向追溯 EK/Evidence。

## A 组 · Harness 底座（models / config / cli / factories）

**EK-01 code-as-action 决策范式（L2）**：模型输出 strict JSON 的 `bash_command`/`python_code` 字段，环境以 subprocess 执行并把 observation 回流——动作空间是**可执行代码**而非点击坐标。证据：`models/base.py#_query_async`、`environments/local_workspace.py#execute`、`config/base.yaml#system_template`（"Emit exactly ONE JSON object per turn… bash_command: <exactly one shell command>"）。links: mechanism(EK-06,EK-07,EK-36)；contrast(EK-46)；subsystem(agents/environments)。

**EK-02 Observation 六要素契约（L1）**：每次执行后捕获 success/exception/command/returncode/url/title/aria_snapshot/console_output/screenshot_path/workspace_files，截断输出（12000/24000）防止 observation 爆炸。证据：`local_browser.py#_capture_observation`、`local_workspace.py#_capture_observation`。links: mechanism(EK-03)；dependency(EK-01)。

**EK-03 浏览器三模式 + CDP 探活自启（L2）**：local_cdp 先探测 `/json/version`+`/json/list`，找不到目标页才自启 Chrome 并 PUT `/json/new?about:blank`；local_launch 直接 launch；local_persistent 复用持久 context。证据：`environments/local_browser.py#_local_cdp_browser`、`_ensure_browser_async`。links: mechanism(EK-05,EK-51)；subsystem(environments)。

**EK-04 python_code 注入执行（L2）**：模型生成的 Python 代码被包装为 `async def __agent_step__(page,context,browser,playwright,task)` 后 exec，`asyncio.wait_for` 超时（默认 30s），观察由页面对象集合计算。证据：`local_browser.py#_execute_async`。links: mechanism(EK-01)；dependency(EK-03)；constraint(EK-55)。

**EK-05 无代理本地 CDP 访问（L1）**：访问 localhost CDP 端点时用 `ProxyHandler({})` 构造无代理 opener，避免系统代理劫持本机调试端口。证据：`local_browser.py#_LOCAL_CDP_OPENER`。links: mechanism(EK-03)；constraint(EK-04)。

**EK-06 bash 语法预检（L1）**：`_validate_bash_command` 用 `["/bin/bash","-n"]` 干跑预检，语法错误抛 FormatError（在花费 subprocess 成本前拒绝）。证据：`models/base.py#_validate_bash_command`。links: mechanism(EK-01)；causal→(EK-07)。

**EK-07 strict JSON schema + done 降级（L2）**：`_response_schema` 为 strict 四字段（thought/<action>/done/final_response，additionalProperties:false）；`parse_json_output` 中"done=true 同时非空 action"自动降级 done=false——strict-schema 产生方不兼容非 strict 消费方时的容忍。证据：`models/base.py#_response_schema`、`parse_json_output`。links: mechanism(EK-01,EK-08)；causal→(EK-17)。

**EK-08 JSON 解析修复循环（L1）**：MAX_JSON_PARSE_RETRIES=3；解析失败把错误与原文回灌模型要求重发，仍失败抛 FormatError。证据：`models/base.py#_query_async` retry 段。links: mechanism(EK-07)；causal→(EK-20)。

**EK-09 重试分类学（L2）**：`_is_rate_limit_error`（429 或 rate-limit 文本，沿 `__cause__` 链）与 `_is_transient_http_error`（httpx Timeout/Network/RemoteProtocol + 408/409/425/500/502/503/504 + 文本匹配）分开退避：rate_limit `min(5*(attempt+1),30)`、transient `min(2*(n+1),10)`；fatal 直接抛。证据：`models/base.py#_post_with_retries`、`_rate_limit_backoff`、`_transient_backoff`。links: mechanism(EK-01)；dependency(EK-12)。

**EK-10 观测模板 + 截图附件开关（L1）**：observation 经 jinja2 observation_template 渲染回流；`attach_observation_screenshot`（默认 false）控制是否 base64 附截图——token 预算显式化。证据：`models/base.py#format_observation_messages`、`config/base.yaml#model.attach_observation_screenshot`。links: mechanism(EK-02)；constraint(EK-11)。

**EK-11 usage/request 双轨指标（L1）**：每请求记录 usage（input/output/total/cached/reasoning）与 request metrics，last+cumulative 双轨累计，随轨迹持久化。证据：`models/base.py#_query_async`、`models/openai_model.py#_usage_metrics_from_response_payload`。links: mechanism(EK-09)；subsystem(models)。

**EK-12 序列化脱敏（L1）**：`serialize()` 将 API key 替换为 `"<redacted>"`；runtime log 经 append_runtime_log 落盘；轨迹消息磁盘化前经 `_sanitize_message_for_disk`（剥 input_image）。证据：`models/base.py#serialize`、`agents/default.py#_sanitize_message_for_disk`、`utils/logging.py`。links: mechanism(EK-09,EK-19)；constraint(EK-57)。

**EK-13 配置栈式合并（L2）**：`-c` 可重复（默认 `["base.yaml","model_openai.yaml"]`）；`get_config_from_spec` 支持本地路径/builtin/inline `key=value`（点分嵌套，yaml.safe_load 值）；`recursive_merge` 用 UNSET 哨兵忽略、dict 递归、后者覆盖前者。证据：`config/__init__.py#get_config_from_spec`、`utils/serialize.py#recursive_merge`、`run/cli.py#DEFAULT_CONFIGS`。links: mechanism(EK-14,EK-39)；subsystem(run/config)。

**EK-14 config_snapshot 可复现性（L2）**：每次 run 落盘 `config_snapshot/`（manifest + 各 spec 副本 + merged_config.yaml）；工具（self_reflection/image_qa）经 `_model_config.load_tool_model` 读 merged_config 复用与 agent 同一模型。证据：`config/__init__.py#snapshot_config_specs`、`tools/_model_config.py`。links: mechanism(EK-13)；causal→(EK-49)。

**EK-15 工厂映射表（L1）**：get_agent/get_environment/get_model 用 dict 映射 + 模块路径 spec（`webwright.agents.default.DefaultAgent`），deepcopy config 后构造。证据：`agents/__init__.py`、`environments/__init__.py`、`models/__init__.py`。links: mechanism(EK-16)；subsystem(webwright)。

**EK-16 Protocol 接口（L1）**：`__init__.py` 用 typing.Protocol 定义 Model/Environment/Agent 最小接口（Model: __call__/query/format_message/format_observation_messages/get_template_vars/serialize）。证据：`webwright/__init__.py#Model`。links: mechanism(EK-15)；constraint(EK-01)。

**EK-17 完成门控：self_reflection 结果硬性放行（L2）**：`require_self_reflection_success` 开启时，done 放行需 `final_runs/run_<latest>/self_reflect_result.json` 的 `predicted_label==1`；否则 `_self_reflection_gate_error` 打断。**精度注（Reconciliation R1）**：代码默认 `require_self_reflection_success=False`（agents/default.py#AgentConfig:39），base.yaml 配置层开启 `true`——与 step_limit（代码默认 15 / 配置 100）同属"配置覆盖代码默认"模式，默认值声明均以配置生效值计。证据：`agents/default.py#_self_reflection_gate_error`、`_tool_gate_error`、`config/base.yaml#agent.require_self_reflection_success`。links: causal→(EK-49)；dependency(EK-53)；constraint(EK-07)。

**EK-18 上下文压缩保护（L1）**：`_compact_history` 失败不终止 run（压缩是优化不是正确性依赖）；keep_last_n_observations 限制 observation 回看。证据：`agents/default.py#_compact_history`。links: mechanism(EK-02)；subsystem(agents)。

**EK-19 轨迹磁盘化脱敏（L1）**：写盘消息剥除 input_image 与敏感负载，磁盘轨迹与内存消息分离。证据：`agents/default.py#_sanitize_message_for_disk`。links: mechanism(EK-12)；constraint(EK-17)。

**EK-20 异常→exit_status + close 分离（L1）**：run 异常与 close 异常分开捕获，exit_status=type(exc).__name__（LimitsExceeded/Submitted/FormatError/异常类名）；close 失败也 re-raise（不吞）。证据：`run/cli.py#run_one`。links: causal→(EK-08)；subsystem(run)。

**EK-21 时间戳输出目录（L1）**：`<task_id>_<YYYYmmdd_HHMMSS>` 命名 run 目录，天然不覆盖历史 run。证据：`run/cli.py#_timestamped_output_dir`。links: mechanism(EK-13)；subsystem(run)。

**EK-22 debug 模式可观测性（L1）**：debug=headed+devtools+keep_open+prompt_before_close+slow_mo 250，失败可人工介入。证据：`run/cli.py` debug 分支。links: mechanism(EK-14)；constraint(EK-20)。

## B 组 · Skill Factory 蒸馏（learn/build/update/init）

**EK-23 learn 管道（L2）**：collect_runs→gate 过滤错 solve→模板分组→canonicalize 答案→evolve 入库。证据：`skill_factory/learn.py#run`、`docs/skill_factory/manual.md`。links: causal→(EK-24,EK-26,EK-27,EK-35)；subsystem(skill_factory)。

**EK-24 gate 准入三态（L2）**：gold（=== 比对）/self_verify（非空+shape 匹配+agent 自报 SUCCESS）/none；**自述局限：self_verify 挡不住 agent 误信的错答案**；升级路径=WebJudge 或跨源一致性。证据：`skill_factory/gate.py`。links: causal→(EK-25,EK-28)；contrast(EK-17)。

**EK-25 答案恢复三态回退（L1）**：`_recover_answer` ①agent_response.json 的 retrieved_data → ②trajectory exit message（仅 exit_status=="Submitted" 且非空 final_response 且 JSON-ish）→ ③skip+human reason。**精度注（Reconciliation R2）**：webwright 主 solve **不写** agent_response.json（那是 skill_factory 执行器产物）；主场景答案来自 trajectory exit message 的 `extra.submission/extra.final_response`（始终 STRING，仅任务要求格式时才结构化），①是兼容 skill_factory 管道的分支。证据：`skill_factory/learn.py#_recover_answer`（含 87-90 行注释）。links: dependency(EK-24)；mechanism(EK-36)。

**EK-26 模板分组规则（L2）**：`_GROUP_SYS` 规定参数/措辞/过滤条件可合并；**换站/换核心动作/换优化目标/换 output schema 必须拆**。证据：`skill_factory/learn.py#_GROUP_SYS`。links: causal→(EK-35)；constraint(EK-34)。

**EK-27 canonicalize_answers（L1）**：机械快路径（全结构化且同 shape→无模型）否则单次 LLM "reshape-only 不新增数据"。证据：`skill_factory/learn.py#canonicalize_answers`。links: mechanism(EK-23)；constraint(EK-42)。

**EK-28 _norm 归一化（L2）**：折叠"写法"不折叠"内容"：大小写、空格、引号、`5=="5"` 类型抖动折叠；`"AS 26"=="AS26"`；但 AS27≠AS26、`"\b 434"≠"B6 434"`（regex 转义 bug 不被洗白）。证据：`skill_factory/update.py#_norm`、`tests/skill_factory/test_evolve.py#test_norm_*`。links: mechanism(EK-29,EK-24)；subsystem(update)。

**EK-29 无模型重放（L2）**：`_replay` 在临时目录 `python skill.py taskspec.json`（SKILL_RUN_TIMEOUT 240s），读 agent_response.json 比对；replay 是技能入库的最终裁判。证据：`skill_factory/update.py#_replay`、`skill_factory/execute.py`。links: dependency(EK-28)；causal→(EK-31,EK-32)。

**EK-30 双层预算 draws×rounds（L2）**：draws=独立蒸馏尝试（注释测得 ~40% 首抽即过），rounds=同一候选的修复反馈轮；任一层验证通过即停——**重画优于无脑修复**。证据：`skill_factory/update.py#_refine`、`tests/skill_factory/test_evolve.py#test_a_fresh_draw_lands_where_repairing_the_bad_one_would_not`。links: mechanism(EK-31)；contrast(EK-30a)。

**EK-31 grade 三态语义（L1）**：executable（replay 复现）/ reference（replay 失败但留可读 prior）/ unverified（verify off 从未跑）——**"没测过"≠"测过但失败"**，三态严格区分。证据：`skill_factory/update.py` grade 逻辑、`tests/test_evolve.py#test_every_skill_carries_a_grade_and_the_three_states_are_distinct`。links: mechanism(EK-30,EK-32)；constraint(EK-43)。

**EK-32 永不覆盖已验证技能（L1）**：on_fail=reference 时 refine 失败绝不动已有 executable 技能（"GOOD OLD CODE 存活"测试）；对"VERIFIED 但无 replays.json"的既有技能 refine 直接跳过。证据：`skill_factory/update.py#_refine` guard、`tests/test_evolve.py`（"old verified skill must survive"）。links: constraint(EK-31,EK-57)；mechanism(EK-33)。

**EK-33 增量 refine regression-replay（L2）**：refine 时把旧 replays.json 例并入 replay 集做回归——只满足新实例、打破旧 coverage 的候选被拒（[42] 存活、[43] 不落地）。证据：`skill_factory/update.py#_refine` replay 集构造、`tests/test_evolve.py`（"refine breaking old coverage must be rejected"）。links: dependency(EK-32)；causal→(EK-35)。

**EK-34 _memorized_answer 反模式检测（L2）**：脚本内**逐字包含全部答案字段**判定为"认识答案而非提取"，丢弃（实测 3/3 clean solves 也部分命中、0/5 distilled 命中、1/1 失败案例命中——宽判据误杀，窄判据正确）；单字段出现（如 "United" 是词汇表）不触发。证据：`skill_factory/update.py#_memorized_answer`、`tests/test_evolve.py#test_a_solve_that_recognises_its_answer_is_not_material`。links: constraint(EK-26,EK-54)；mechanism(EK-28)。

**EK-35 evolve changelog（L1）**：`{"added": [...], "adapt_refined": [...], "use": [...], "dropped_wrong": n, "dropped_lookup": n, ...}`——静默丢弃=谜团，changelog 与日志都点名。证据：`skill_factory/update.py#evolve` 返回、`tests/test_evolve.py#test_evolve_drops_a_lookup_and_says_so`。links: mechanism(EK-23,EK-33)；subsystem(update)。

**EK-36 answer 文件是契约，非退出码（L2）**：solve 写了 agent_response.json（retrieved_data）即使非零退出也计 solved；退出码只是进程健康。证据：`skill_factory/build.py#_already_solved` 与 solve 判定、`tests/test_build_init.py#test_a_solve_that_wrote_its_answer_counts_even_if_it_exited_non_zero`。links: mechanism(EK-01,EK-25)；subsystem(build)。

**EK-37 build 并行 + ticker（L1）**：as_completed 而非 map（map 让完成实例在慢实例后不可见、像卡死）+ 30s 进度 ticker。证据：`skill_factory/build.py` 并行段。links: mechanism(EK-38)；subsystem(build)。

**EK-38 _already_solved 断点续跑（L1）**：任务文本匹配 + agent_response.json 存在 → 跳过重解（幂等/可恢复）。证据：`skill_factory/build.py#_already_solved`、`tests/test_build_init.py#test_resume_finds_a_prior_run_that_produced_an_answer`。links: mechanism(EK-37)；constraint(EK-36)。

**EK-39 agent_cfg 自动补 base.yaml（L1）**：`-c REPLACES` 默认 → 路由自动 re-add base.yaml（除非已含）；OPENAI_ENDPOINT/OPENAI_MODEL 环境存在时自动构造配置零文件（防 gateway 401）；SKILL_AGENT_MAX_TOKENS 默认 16000 覆盖 base.yaml 4000（大技能不被截断）。证据：`skill_factory/route.py#_agent_cfg` 注释。links: mechanism(EK-13,EK-40)；subsystem(route)。

**EK-40 机器相关配置留 CLI 不进 spec（L1）**：build 的 `-c/--jobs/--library` 属 CLI；spec.yaml 保持可提交（policy 块全有默认）。证据：`skill_factory/build.py`、`docs/manual.md`。links: constraint(EK-39)；mechanism(EK-13)。

**EK-41 init drift 预判（L2）**：**判定标准=明天重跑同一任务答案是否仍同串**（价/库存/排行→drift→verify:shape；航班表/规格/静态文本→strict）；账号作用域无法给真实实例→instances:[] 拒绝编造；无洞可变化→拒绝并引导 "say what CHANGES between runs"。证据：`skill_factory/init.py#_SYS`、`tests/test_build_init.py#test_init_picks_the_verify_mode_from_whether_the_answer_drifts`。links: causal→(EK-42,EK-29)；subsystem(init)。

**EK-42 fill_params 不猜（L2）**：只读任务文本提取槽位；未声明的值回 null 而非发明——"wrong-but-plausible value is worse than null"（自信的错答案比空更危险）。证据：`skill_factory/fill.py#_SYS`、`tests/test_fill.py`。links: constraint(EK-43,EK-27)；mechanism(EK-44)。

**EK-43 promote 三闸（L1）**：grade==executable + template_gap 3a 双向检查（任务多要/模板强加）+ 槽位全填充才 run；任何一闸不过降 adapt（"won't guess"）。证据：`skill_factory/decide.py#promote`。links: dependency(EK-42,EK-31)；causal→(EK-44)。

**EK-44 route run/adapt/skip + fallback（L2）**：run（executable+覆盖+槽全填→直接跑，well-shaped 即答不启 agent）；adapt（复用核心/改最后一步→带 hint 交 agent）；skip（从零）；**run 失败或形状不合格→作为 adapt 回退 agent，hint 带 "directly was tried and failed"**（fallback 显式标记，非静默）。证据：`skill_factory/route.py#route`、`tests/test_route.py#test_run_failure_falls_back_to_agent_marked`。links: causal→(EK-45,EK-39)；mechanism(EK-43)。

**EK-45 recommend 护栏（L1）**：decide 返回的 skill_id 必须 ∈ retrieved 候选（防 LLM 幻觉 id）；缺失/空库大声报错（不静默 skip）；`_how_to_reuse` 按 grade 诚实指引（run→direct run；executable→read ENTIRE source；reference→只当 prior 重写最后一步）；异常→降级 skip 但带 "LOOKUP FAILED" 响亮错误。证据：`skill_factory/tools_skill_use.py`（tools/skill_use.py）。links: constraint(EK-44,EK-46)；mechanism(EK-05b)。

**EK-46 retrieve LLM relevance vs simple 回退（L1）**：默认 LLM catalog 全量注入（k=3）；库大时换 embeddings 但**接口不变**；简单关键词 overlap 作 no-LLM 回退。证据：`skill_factory/retrieve.py`。links: mechanism(EK-45)；contrast(EK-01)。

**EK-47 entry_shim 双入口等价（L1）**：`skill.py taskspec.json` 原样通过（replay 字节不变）；`--flags` 组 taskspec 写临时文件重指 argv；无参/--help 打印参数+示例（无 IndexError）；幂等（含 `_skillfactory_cli()` 标记不叠加）。证据：`skill_factory/entry_shim.py#prepend_cli_shim`、`tests/skill_factory/test_entry_shim.py`。links: mechanism(EK-29)；subsystem(entry_shim)。

**EK-48 _slug 防碰撞（L1）**：48 字符截断 + 确定性 disambiguation（长模板共享前缀不撞 id；短模板保留可读 slug）。证据：`skill_factory/update.py#_slug`、`tests/test_evolve.py#test_slug`。links: mechanism(EK-35)；subsystem(library)。

## C 组 · 工具与插件（self_reflection / persistent browser / 插件层）

**EK-49 两阶段截图裁判（L2）**：Stage1 逐图 `Score: 1-5`+`Reasoning`（重试 3 次，ParseFailed 记 0 不炸整 run）；Stage2 全部 reasonings+action_history_log(final_script_log.txt) 汇入终审，要求末尾 `Status: success/failure`（取最后一个匹配，缺失=FAIL exit 1）；输出 predicted_label 1/0/null。证据：`tools/self_reflection.py`。links: causal→(EK-17)；dependency(EK-14,EK-53)。

**EK-50 auto-discover 最新 run（L1）**：用最高编号 `final_runs/run_<n>/screenshots` 自动附加，无需显式图片列表（base.yaml instance_template 推荐用法）。证据：`tools/self_reflection.py` auto 分支、`config/base.yaml`。links: mechanism(EK-49,EK-53)。

**EK-51 持久浏览器会话管理（L1）**：`--remote-debugging-port=0`+独立 user-data-dir 拉 detached headless Chromium（start_new_session）；解析 stderr `DevTools listening on ws://...`；session JSON {id,pid,connectUrl,userDataDir}；后续 `connect_over_cdp` + **disconnect 而非 close()**（保持跨步存活）；release=SIGTERM→SIGKILL+可选删目录。证据：`tools/persistent_local_browser.py`。links: mechanism(EK-03)；contrast(EK-04)。

**EK-52 插件层宿主原生能力替换（L2）**：Claude Code 版 SKILL.md 用宿主 Read/Bash 原生能力**替换** image_qa/self_reflection（"you read PNGs with Read and verify success against plan.md yourself. No OPENAI_API_KEY required"）——同一 workspace contract，不同验证执行者。证据：`skills/webwright/SKILL.md`。links: contrast(EK-49)；mechanism(EK-53)。

**EK-53 workspace 契约（L2）**：plan.md 关键点清单（每个 CP 必须可由截图/日志独立验证）+ final_runs/run_<id>/{final_script.py, final_script_log.txt, screenshots/final_execution_<n>_<action>.png} + self_reflect_config.json 四提示词；final_script.py 仪表化（step <n> action 行 + 最终 datum 打印）。证据：`config/base.yaml#instance_template`、`skills/webwright/SKILL.md`。links: dependency(EK-49,EK-17)；subsystem(harness/plugin)。

**EK-54 提示词防泄漏（L1）**：learn 的 collect_runs 剥离 task.json 中 "## Skill library" 段与 "Additionally, write the final answer into" 指令（F7——pipeline 文本不得泄漏进技能模板）。证据：`skill_factory/learn.py` 清理段、`tests/skill_factory/test_learned_example.py`（"pipeline text must not leak"）。links: constraint(EK-34,EK-26)；mechanism(EK-47)。

**EK-55 cwd 沙箱（L1）**：local_workspace `_resolve_cwd` 强制 resolve 后 relative_to(workspace_dir)，越界抛 ValueError("Command cwd must stay inside workspace")。证据：`environments/local_workspace.py#_resolve_cwd`。links: constraint(EK-04,EK-57)；mechanism(EK-01)。

**EK-56 manifest admit 必须 bool（L1）**：traces_from_manifest 对缺失 admit 抛 KeyError、对字符串 "false"（truthy）抛 TypeError——**默认拒绝，绝不默认 admit**。证据：`skill_factory/update.py#traces_from_manifest`、`tests/test_evolve.py`。links: constraint(EK-35,EK-57)；mechanism(EK-23)。

**EK-57 credentials 语义（L1）**：manual mode（update）manifest 每 run 携带 credentials 进 replay（登录站可验证）；learn 不携带；credentials **永不写入 skill/replays.json**（库保持可共享提交）。证据：`docs/skill_factory/manual.md`、`skill_factory/update.py` replay 环境构造。links: constraint(EK-32,EK-55,EK-12)；subsystem(update).

---
## EK Graph 汇总
- 平均出边：48 条 EK、96 条边引用（links 计数含多边条目）——平均 ≥1 ✅
- 游离 EK（无 links）：0 ✅
- 边类型覆盖：mechanism 24 / subsystem 14 / causal 13 / dependency 10 / constraint 14 / contrast 6（含跨条目双向引用）✅
- 聚合规则覆盖率（见 03 层）：8/8 KO 声明 R1-R4 ✅
