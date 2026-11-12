# Flow Atlas — 七类流

> 每条 Edge 均可回溯到 symbol / file / condition。标记：`main` 主路径，`alt` 分支，`fail` 失败路径，`bypass` 绕过/覆盖路径。
> 代码基准：commit `6035d56`。

## F1 Control Flow（控制流）

| Edge | From → To | Symbol / File:Line | Condition | 类型 |
|---|---|---|---|---|
| F1-E1 | CLI → MemoryEvolver | `evolve_cli.py` `cmd_analyze/generate/auto` → `MemoryEvolver.run_phase` | 子命令匹配 | main |
| F1-E2 | MemoryEvolver → PhaseAnalyzer | `MemoryEvolver.analyze` → `PhaseAnalyzer.run_analysis`（`memory_evolver.py:202-296`） | `phases.analyze.completed == False` | main |
| F1-E3 | PhaseAnalyzer → AnalysisAgent | `phase_analyzer.py:43-91` → `base_agent.py:134-159` | 有 task_logs | main |
| F1-E4 | MemoryEvolver → PhaseGenerator | `generate:202-296` | analyze 完成 | main |
| F1-E5 | MemoryEvolver → MemoryCreator | `create:298-381` → `memory_creator.py:86-284` | generate 完成且 provider_code 非空 | main |
| F1-E6 | MemoryEvolver → PhaseValidator | `validate:383-473` → `phase_validator.py` | create 完成 | main |
| F1-E7 | AutoEvolver → run_provider × N | `auto_evolver.py:894-918` → `run_provider.py:23-86` | 每轮 | main |
| F1-E8 | Runner → ToolCallingAgent | `run_flash_searcher_mm_gaia.py:255-271` → `FlashOAgents/agents.py` | `--memory_provider` 存在 | main |
| F1-E9 | Agent planning → provide_memory(BEGIN) | `agents.py:414` | 每次任务 | main |
| F1-E10 | Agent step → provide_memory(IN) | `agents.py:783` | 每个 step | main |
| F1-E11 | Agent 结束 → take_in_memory | `run_flash_searcher_mm_gaia.py:319-338` | 正常返回 | main |
| F1-E12 | Agent 异常 → take_in_memory(error) | `run_..._gaia.py:369-388` | 捕获异常 | fail |
| F1-E13 | Validator 失败 → SWEAgentValidator.fix | `phase_validator.py:197-214` | `enable_auto_fix=True` | alt |
| F1-E14 | 修复通过 → sync 真实目录 | `phase_validator.py:814-835` | verdict=passed | bypass（改写真实文件） |
| F1-E15 | Creator 配置校验失败 → warning 继续 | `memory_evolver.py:266-271` | `_validate_config_updates` 异常 | bypass |

**控制流关键判断**：元进化的“控制”不是 API 调用链，而是**代码写入链**；F1-E14 是唯一的跨层写回真实环境路径。

## F2 State Flow（状态流）

| Edge | From → To | Symbol / File:Line | Condition | 类型 |
|---|---|---|---|---|
| F2-E1 | Request → provider | `MemoryStatus.BEGIN/IN`（`memory_types.py:11-15`） | 阶段 | main |
| F2-E2 | 任务 → 短期快照 | `_task_snapshots[sha256(query)]`（`lightweight:501-556`） | Lightweight | main |
| F2-E3 | step → context delta | `lightweight:558-582` | current 以 last 为前缀 | alt |
| F2-E4 | step → 整段 context | `lightweight:558-582` | 非前缀 | alt |
| F2-E5 | 短期超限 → LLM 选择 | `lightweight:799-920` | `len > max_shortterm_items` | main |
| F2-E6 | LLM 失败 → FIFO | `lightweight:799-920` | 异常 | fail |
| F2-E7 | 长期超限 → LLM 删除 | `lightweight:1159-1341` | `len > max+buffer` | main |
| F2-E8 | 长期 LLM 失败 → 分数排序 | `lightweight:1159-1341` | 异常 | fail |
| F2-E9 | 每 50 任务 → consolidation | `cerebra:1044-1048` | `task_counter % 50 == 0` | main |
| F2-E10 | 每轮 → storage 备份 | `auto_evolver.py:812-840` | 新轮开始 | main |
| F2-E11 | 阶段完成 → state.json | `memory_evolver.py:1-150` | 每阶段结束 | main |
| F2-E12 | 每步 → checkpoint | `auto_evolver.py:788-1008` | 每轮结束 | main |

**状态流关键判断**：状态没有统一状态机；F2-E2 的 query-SHA 键与 F2-E7 的索引位移共同构成 KO-05 的候选缺陷来源（EK-32）。

## F3 Data Flow（数据流）

| Edge | From → To | Symbol / File:Line | Condition | 类型 |
|---|---|---|---|---|
| F3-E1 | 数据集 → runner | `MemEvolve/config.py:16-31` | 数据集名 | main |
| F3-E2 | task → agent messages | `agents.py:408-425` | 任务开始 | main |
| F3-E3 | trajectory → TrajectoryData | `run_..._gaia.py:321-331` | 任务结束 | main |
| F3-E4 | TrajectoryData → provider.take_in | `base_memory.py:31` | 调用 | main |
| F3-E5 | trajectory → LLM summary | `agent_kb:603-656` / `memp:229-258` | provider 各异 | main |
| F3-E6 | summary → JSON | 各 provider `json.dump`（E-090） | 写入 | main |
| F3-E7 | JSON → embedding 索引 | `agent_kb:116-146`；`evolver:123-159` | initialize/写入后 | main |
| F3-E8 | 检索结果 → prompt | `agents.py:413-425` / `782-795` | 有返回 | main |
| F3-E9 | 工具记忆 → callable | `cerebra:955-981`；`skillweaver:182-195` | type=API | alt |
| F3-E10 | 生成代码 → provider 文件 | `memory_creator.py:86-284` | create | main |

## F4 Evidence Flow（证据流）

| Edge | From → To | Symbol / File:Line | Condition | 类型 |
|---|---|---|---|---|
| F4-E1 | 任务结果 → judge | `run_..._gaia.py:305-315` | judge_model 存在 | main |
| F4-E2 | judge 异常 → is_correct=False | `run_..._gaia.py:316-317` | 异常 | fail |
| F4-E3 | 任务日志 → 指标 | `trajectory_tools.py:116-230` | 批次结束 | main |
| F4-E4 | 指标 → AutoEvolver 选择 | `auto_evolver.py:623-782` | 评估完成 | main |
| F4-E5 | 隔离运行日志 → verdict | `phase_validator.py:565-581/762-789` | 出现 `[ERROR]` | main |
| F4-E6 | verdict → 真实环境回写 | `phase_validator.py:814-835` | passed + auto-fix | bypass |
| F4-E7 | winner → state/checkpoint | `auto_evolver.py:920-945` | 每轮结束 | main |
| F4-E8 | 论文数字 → README | `README.md` | 人工撰写 | S0-vendor（不进入本考古证据链的 Fact） |

**证据流缺口**：F4-E5 的“证据”是日志字符串包含 `[ERROR]`，不是结构化断言；F4-E3 只给均值，无样本级置信信息（EK-50）。

## F5 Authority Flow（权威流）

| Edge | From → To | Symbol / File:Line | Condition | 类型 |
|---|---|---|---|---|
| F5-E1 | 枚举值 → 类/模块 | `PROVIDER_MAPPING`（`memory_types.py:48-63`） | MemoryType(value) | main |
| F5-E2 | 类/模块 → import | `phase_analyzer.py:15`；`phase_validator.py:433-790` | 动态 import | main |
| F5-E3 | Creator → enum/mapping | `memory_creator.py:86-284`（锚点） | create | main |
| F5-E4 | Creator → config.py | `memory_creator.py:86-284`（锚点） | config_updates 合法 | main |
| F5-E5 | Validator → 真实 provider.py | `phase_validator.py:814-835`（copy2） | passed | bypass |
| F5-E6 | 静态检查 → 通过 | `phase_validator.py:224-293`（子串） | 字符串存在 | alt（弱权威） |
| F5-E7 | AutoEvolver → storage 重置 | `auto_evolver.py:812-840` | 每轮 | main |

**权威流关键判断**：权威来源是**字符串锚点 + 文件系统写权限**，不是类型系统或 schema；F5-E6 是“弱权威”路径，可能让仅含注释的实现通过。

## F6 Memory Flow（记忆流）

| Edge | From → To | Symbol / File:Line | Condition | 类型 |
|---|---|---|---|---|
| F6-E1 | 轨迹 → 经验条目 | `agent_kb:603-656`；`expel:220-247` | 成功门控 | main |
| F6-E2 | 条目 → 合并 | `evolver:678-683` | cosine>0.8 且 LLM 判同 | alt |
| F6-E3 | 条目 → 新建 | `evolver:636-688` | 不满足合并 | alt |
| F6-E4 | 失败 → 修订旧条目 | `memp:392-476` | is_correct False | alt |
| F6-E5 | 相似 query → 重写 cheatsheet | `dynamic:293-321` | IN + top1 | main |
| F6-E6 | 节点 → 邻居扩展 | `cerebra:493-531` | 语义边≥0.75，≤3 跳 | alt |
| F6-E7 | 工具代码 → 安全检查 → 注册 | `cerebra:1232-1355/1570-1585` | 抽取成功 | main |
| F6-E8 | 低分/超容 → 剪枝 | `evolver:412-435`；`lightweight:1159-1341` | 阈值/容量 | main |
| F6-E9 | 技能 → Python 文件 | `skillweaver:41` | 双成功门控 | main |

## F7 Policy Flow（策略/治理流）

| Edge | From → To | Symbol / File:Line | Condition | 类型 |
|---|---|---|---|---|
| F7-E1 | 成功判定策略 → 写入 | `agent_kb:663-679`；`memp:480`；`evolver:272` | provider 级默认 | main |
| F7-E2 | 配置校验策略 → 继续 | `memory_evolver.py:266-271` | 非法 config | bypass |
| F7-E3 | 自动修复策略 → 修复+回写 | `phase_validator.py:29-43/814-835` | 默认 True | bypass |
| F7-E4 | 公平性策略 → storage 重置 | `auto_evolver.py:812-840` | 每轮 | main |
| F7-E5 | 并发策略 → concurrency=1 | `run_provider.py:48-63` | 固定 | main |
| F7-E6 | 超时策略 → 无 | `run_provider.py:70-78`（`wait()` 无 timeout） | 无 | 缺口 |
| F7-E7 | 测试策略 → 无核心测试 | `find`（E-091） | 无 | 缺口 |

**策略流关键判断**：项目没有集中式 policy 文件；策略以**构造函数默认值 + 注释 + 文档字符串**分散存在，导致 F7-E2/E3 两条 bypass 路径无法被统一审计。

## 交叉校验（Flow → KO）
| KO | 依赖的 Flow Edge | 是否一致 |
|---|---|---|
| KO-01 | F6-E1/E2/E4/E5 + F3-E6 | 一致 |
| KO-02 | F1-E9/E10 + F2-E1 | 一致 |
| KO-03 | F7-E1 + F4-E1/E2 | 一致 |
| KO-04 | F6-E6/E8 + F5-E1 | 一致 |
| KO-05 | F2-E2/E3/E7 + F2-E10 | 一致 |
| KO-06 | F1-E5/E15 + F3-E10 + F5-E3 | 一致 |
| KO-07 | F4-E5/E6 + F5-E5/E6 + F7-E3 | 一致 |
| KO-08 | F2-E10/E12 + F4-E4/E7 | 一致 |
| KO-09 | F3-E9 + F6-E7/E9 + F5-E1 | 一致 |
