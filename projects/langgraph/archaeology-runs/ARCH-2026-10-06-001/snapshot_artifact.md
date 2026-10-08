# Snapshot Artifact — ARCH-2026-10-06-001 · langgraph

## 1. Repository Snapshot
| 字段 | 值 |
|---|---|
| repository | https://github.com/langchain-ai/langgraph.git |
| commit SHA | `2d942085e214ef6b99b6f54ed4d544a7c7c5ac56` |
| branch | main（浅克隆 depth 1） |
| commit 时间 | 2026-10-05 13:46:44 -0400 |
| commit 标题 | fix(langgraph): fork before replaying an update checkpoint the thread moved past (#9170) |
| version | langgraph 1.2.13（libs/langgraph/pyproject.toml） |
| license | MIT |
| snapshot 时间 | 2026-10-06（UTC+8） |
| clone 方式 | `git clone --depth 1`（只读） |

## 2. 项目基础地图
### 2.1 一句话定位
LangGraph = 低层有状态多 Actor Agent 编排框架（"Low-level orchestration framework for building stateful agents"）：把 agent 定义为**有状态图**，节点=执行单元、边=状态转换、checkpoint=持久化、interrupt=人机交互挂起/恢复。官方背书企业用户 Klarna/Replit/Elastic。

### 2.2 语言与工程形态
- 主实现：Python（monorepo 亦含 TS/JS SDK）
- Python 核心（libs/ 各库，不含 tests）≈ 84,782 行；含 tests ≈ 193,253 行（wc -l 实测）
- Monorepo：libs/ 下 9 库

### 2.3 仓库结构与入口
```
libs/
├── checkpoint/            # CheckpointSaver 抽象（base/memory/serde）——持久化接口层
├── checkpoint-postgres/   # Postgres 实现
├── checkpoint-sqlite/     # SQLite 实现
├── checkpoint-conformance/ # 多 checkpointer 实现的一致性测试套件
├── cli/                   # LangGraph CLI（langgraph dev/build）
├── langgraph/             # 核心框架（本次考古主对象）
├── prebuilt/              # 高层 API（create_react_agent 等）
├── sdk-py / sdk-js        # Server REST API 客户端
libs/langgraph/langgraph/
├── graph/                 # 图构造 API（StateGraph/Graph + nodes/edges/compile）
├── pregel/                # 执行引擎（Pregel 式：_algo/_checkpoint/_loop/_retry/_runner/_executor/_io/_read/_write/_call/_tools/_validate）
├── channels/              # 状态通道原语（LastValue/Topic/BinaryOperator/Context）
├── managed/               # Managed State（SharedValue 等跨节点状态）
├── func/                  # 函数式入口（func.execute）
├── _internal/             # 内部运行时（协程/任务调度）
├── runtime.py / config.py / errors.py / types.py / callbacks.py
```
**入口**：`libs/langgraph/langgraph/__init__.py` → graph/state.py（StateGraph）、pregel/main.py（Pregel 类 = 编译后的 Runnable）。

### 2.4 核心数据结构
| 结构 | 文件 | 角色 |
|---|---|---|
| `StateGraph` / `Graph` | graph/state.py, graph/graph.py | 用户侧图定义（节点/边/条件边/共享状态） |
| `Pregel` | pregel/main.py | 编译后执行图（Runnable，含 configurable checkpointer/store） |
| `Channel` 原语 | channels/*.py | 状态聚合语义（LastValue 覆盖 / Topic 累积 / BinaryOperator 归约 / Context） |
| `Checkpoint` | checkpoint/base/checkpoint.py | 线程状态快照（channel_values/versions/ts/metadata） |
| `CheckpointSaver` | checkpoint/base/checkpointer.py | 持久化接口（get/put/list/agets/aputs） |
| `ChannelVersions` | pregel/types.py | 版本计数表（乐观并发与写冲突检测核心） |
| `Task`/`TaskProtocol` | pregel/types.py | 执行单元（writes 累积） |

### 2.5 状态与生命周期
- 线程（thread）粒度状态：每线程一个 checkpoint 链；`thread_id` 为最小持久化单元
- 状态转移：节点执行 → writes 写回 channels → checkpoint 落盘（版本表推进）→ 下一步边解析
- 生命周期：StateGraph.compile(checkpointer=...) → Pregel.invoke/stream → 每超步（super-step）一个 checkpoint；interrupt 挂起 → update_state 续跑
- 错误路径：节点异常 → 按 step 配置的 retry_policy 重试；`None` 返回不写状态；`Command(resume)` 恢复

### 2.6 测试体系
- `libs/langgraph/tests/`：pytest 套件（agents.py/any_int.py/fake_chat.py 等 harness）
- `checkpoint-conformance/`：跨 checkpointer 实现的一致性契约测试（get/put/list/version 语义）
- 工程门禁（AGENTS.md）：任何库改动必须 `make format && make lint && make test`

### 2.7 配置与治理
- AGENTS.md：monorepo 多库规则 + Corridor `analyzePlan` 安全分析（若可用）+ format/lint/test 门禁
- 运行时配置：`config`（thread_id/checkpoint_id/recursion_limit/configurable）贯穿 invoke/stream/update_state
- 权限与治理机制：无内置权限模型（框架层）；CheckpointSaver 为可插拔持久化治理点；interrupt 为 HITL 权威注入点；AGENTS.md 约束代码贡献者

### 2.8 外部依赖（langgraph 核心）
langchain-core≥1.4.7 / langgraph-checkpoint≥4.1.0 / langgraph-sdk≥0.4.2 / langgraph-prebuilt≥1.1.0 / xxhash≥3.5.0 / pydantic≥2.7.4

## 3. 考古范围声明
- 主对象：`libs/langgraph/langgraph/`（graph + pregel + channels + managed + checkpoint 接口）+ `libs/checkpoint/`（持久化契约）
- 深读锚点：pregel/_loop.py（主循环）、pregel/_algo.py（超步调度）、pregel/_checkpoint.py（checkpoint 写入）、pregel/_retry.py（重试）、graph/state.py（通道推导）、channels/*.py（语义原语）、checkpoint/base/*.py（持久化契约）
- 不考古：docs/、examples/、sdk-js（TS）、cli 细节（以核心框架为准）
- 证据引用一律带 `file:symbol` 可回溯
