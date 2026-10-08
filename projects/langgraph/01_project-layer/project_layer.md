# 01 Project Layer — LangGraph 项目地图

## 1. 定位与形态
| 维度 | 事实 | 证据 |
|---|---|---|
| 定位 | 低层有状态多 Actor Agent 编排框架 | libs/langgraph/pyproject.toml description |
| 版本 | 1.2.13 | libs/langgraph/pyproject.toml |
| 语言 | Python（monorepo 含 TS/JS SDK） | libs/ 目录 |
| 规模 | libs Python ≈84,782 行（不含 tests）；含 tests ≈193,253 行 | wc -l 实测 |
| 许可 | MIT | LICENSE / GitHub API |
| 活跃度 | 10-05 仍有修复 commit（HEAD 2d94208） | git log |

## 2. 仓库结构
```
libs/
├── checkpoint/            # CheckpointSaver 抽象（base/memory/serde）+ store（长时记忆）
├── checkpoint-postgres/   # Postgres checkpointer + store 实现
├── checkpoint-sqlite/     # SQLite checkpointer + store 实现
├── checkpoint-conformance/ # 跨 checkpointer 实现一致性契约测试
├── cli/                   # langgraph CLI（dev/build）
├── langgraph/             # 核心框架
├── prebuilt/              # 高层 agent（create_react_agent/tool_node）
├── sdk-py / sdk-js        # server REST 客户端
libs/langgraph/langgraph/
├── graph/                 # StateGraph/Graph（add_node/add_edge/compile）
├── pregel/                # 执行引擎（_loop/_algo/_checkpoint/_retry/_io/_read/_write/_call/_executor/_validate）
├── channels/              # 通道原语（last_value/topic/binop/delta/any_value/ephemeral/untracked/named_barrier）
├── managed/               # ManagedValue（IsLastStep/RemainingSteps）
├── func/                  # 函数式入口
├── _internal/             # 内部运行时（config/constants/runnable/timeout）
├── runtime.py / config.py / errors.py / types.py / callbacks.py
```

## 3. 核心抽象与模块职责
| 抽象 | 文件 | 职责 |
|---|---|---|
| `StateGraph` | graph/state.py:131 | 用户侧图定义；从 schema 推导 channels；compile 产 CompiledStateGraph |
| `Pregel` | pregel/main.py:class | 编译后执行图（Runnable）；invoke/stream/get_state/update_state |
| `PregelLoop` | pregel/_loop.py:162 | 单线程执行循环：tick（调度）→ 执行 → after_tick（写回） |
| `apply_writes` | pregel/_algo.py:232 | 超步写应用：确定性排序→版本推进→通道更新→finish |
| `BaseCheckpointSaver` | libs/checkpoint/langgraph/checkpoint/base/__init__.py:177 | 持久化契约（get/get_tuple/list/put/put_writes + async） |
| `BaseChannel` | channels/base.py:19 | 通道抽象（update/get/checkpoint/from_checkpoint/consume/finish） |
| `BaseStore` | libs/checkpoint/langgraph/store/base/__init__.py:708 | 长时记忆（namespace/key Item、get/search/put/delete/batch） |
| `ManagedValue` | managed/base.py:18 | 跨节点托管状态（is_last_step 等） |

## 4. 生命周期（一次完整执行）
```
StateGraph(State) ──add_node/add_edge──▶ builder
    │ compile(checkpointer=..., interrupt_before/after=...)
    ▼
Pregel（CompiledStateGraph）
    │ invoke(input, {"configurable":{"thread_id":...}})
    ▼
PregelLoop.__enter__ → 读 checkpoint（saver.get_tuple by thread_id）
    ▼ 循环：tick() 每轮
    ├─ step > stop → out_of_steps
    ├─ prepare_next_tasks（按 versions_seen/channel_versions 推导可执行节点）
    ├─ interrupt_before 命中 → raise GraphInterrupt（挂起）
    ├─ 执行 tasks（并发 BackgroundExecutor）
    └─ after_tick() → apply_writes → 新 checkpoint（_put_checkpoint）
    ▼
    tasks 空 → status=done → 输出 values
```
恢复路径：`update_state(config, values, as_node=...)` 把外部值当作某节点的写注入 → 重放；`Command(resume=...)` 恢复 interrupt。

## 5. 关键数据结构
| 结构 | 位置 | 要点 |
|---|---|---|
| `Checkpoint` | checkpoint/base/__init__.py:93 | v/id/ts/channel_values/channel_versions/versions_seen/updated_channels |
| `CheckpointTuple` | 同文件:140 | config+checkpoint+metadata+parent_config+pending_writes |
| `ChannelVersions` | pregel/types.py | 通道版本表（版本字符串或 int） |
| `PregelExecutableTask` | pregel/types.py | 执行单元（path/name/writes/triggers） |
| `Command` | types.py:833 | graph/update/resume/goto 四字段指令 |
| `RetryPolicy` | types.py:427 | initial_interval/backoff_factor/max_interval/max_attempts/jitter/retry_on |

## 6. 状态与配置
- 线程粒度状态：`thread_id` 是 checkpoint 持久化与恢复的主键（BaseCheckpointSaver docstring）
- 运行时配置：`config.configurable`（thread_id/checkpoint_id/recursion_limit/checkpoint_ns）+ `config`（CONFIG_KEY_* 常量，_internal/_constants.py）
- `durability`：checkpoint 落盘时机（"exit" 仅在退出时写 vs 每超步写）——pregel/_loop.py:_put_checkpoint

## 7. 测试体系
- `libs/langgraph/tests/`：pytest（test_pregel 9,856 行 / test_pregel_async 9,718 / test_time_travel 3,966 / test_retry 2,944 / test_delta_channel_* 族）
- `checkpoint-conformance/`：跨 saver 实现一致性契约（put/get/versions 语义）
- 门禁：AGENTS.md 要求 `make format && make lint && make test`

## 8. 配置与治理
- AGENTS.md：monorepo 多库规则 + Corridor `analyzePlan` 安全分析（若可用）+ format/lint/test 门禁
- 框架无内置权限模型；权威注入点是 interrupt（HITL）与 CheckpointSaver（可插拔持久化治理）
- serde 治理：STRICT_MSGPACK 时 compile 构建反序列化白名单（graph/state.py:1233-1254）

## 9. 外部依赖（langgraph 核心包）
langchain-core≥1.4.7 / langgraph-checkpoint≥4.1.0 / langgraph-sdk≥0.4.2 / langgraph-prebuilt≥1.1.0 / xxhash≥3.5.0 / pydantic≥2.7.4（libs/langgraph/pyproject.toml）

## 10. 重要设计与 ADR 证据
- 版本机制：`increment()`（默认版本函数，int+1）——pregel/_algo.py:227；`_uuid5_str/_xxhash_str` 任务 id 生成（_algo.py:1395/1404）
- commit HEAD 语义：`fix(langgraph): fork before replaying an update checkpoint the thread moved past (#9170)` —— 修复 update_state 后旧 checkpoint 分支重放错乱，是版本机制正确性的直接证据
- 近 80 commits（09-01 以来）：57 chore / 14 fix / 5 feat / 3 release——修复集中于 checkpoint/delta/interrupt 语义（#9170/#9165/#9142/#9141/#8548/#8538/#9103）
