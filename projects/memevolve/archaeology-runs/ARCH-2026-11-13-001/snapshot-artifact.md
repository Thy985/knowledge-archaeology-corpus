# Repository Snapshot Artifact — MemEvolve

## 1. 身份与版本

| 项 | 值 | 判定方式 |
|---|---|---|
| Repository | `https://github.com/bingreeky/MemEvolve.git` | job manifest |
| Branch | `main` | `git rev-parse --abbrev-ref HEAD` |
| Commit SHA | `6035d5659d7a092dbfa6a87b1a32a3cee652ba54` | `git rev-parse HEAD` |
| Commit time | `2026-05-05T17:29:28+08:00` | `git log -1 --format=%cI` |
| Commit message | `Update README` | `git log -1 --format=%s` |
| License | Apache-2.0 | `LICENSE` |
| Clone mode | `--depth 1 --single-branch --branch main` | 命令记录 |
| Snapshot time | 2026-11-13 | 本轮运行 |

> 浅克隆只有 1 个 commit，历史深度不可用；本轮所有“历史/修复史”类判断因此受限（见 06 Validation 的 Coverage note）。

## 2. 规模（可复算）

| 指标 | 值 | 命令口径 |
|---|---|---|
| 文件总数（排除 `.git`） | **288** | `find . -path ./.git -prune -o -type f -print \| wc -l` |
| Python 文件 | **146** | `find ... -name '*.py'` |
| Python 总行数 | **33,653** | `find ... -print0 \| xargs -0 cat \| wc -l` |
| 仓库体积 | **175 MB** | `du -sh .` |
| `EvolveLab/` 文件数 | **19** | `find Flash-Searcher-main/EvolveLab -type f` |
| `MemEvolve/` 文件数 | **18**（含 `.DS_Store` 1 个） | `find Flash-Searcher-main/MemEvolve -type f` |
| 13 个 provider 代码行数 | **8,088** | `wc -l EvolveLab/providers/*_provider.py`（不含 template/`__init__`） |
| 最大两个 provider | `cerebra_fusion` 1,601 行；`lightweight` 1,596 行 | 同上 |

## 3. 顶层结构（实际）

```
repo/
├── README.md                 # ICML'26 论文项目说明；明确 "not the final release"
├── LICENSE                   # Apache-2.0
├── requirements.txt
├── evolve_cli.py             # 四阶段 CLI（analyze / generate / create / validate / auto-evolve）
├── base_agent.py             # SearchAgent / MMSearchAgent / AnalysisAgent / ReviewAgent
├── run_flash_searcher_mm_gaia.py / _xbench.py / _webwalkerqa.py   # 数据集 runner
├── eval_utils.py
├── FlashOAgents/             # Agent 运行时（ToolCallingAgent、memory guidance 注入）
├── EvolveLab/                # 记忆抽象层 + 13 providers（19 files）
├── MemEvolve/                # 元进化编排（18 files）
├── mini-swe-agent/           # vendored 第三方（tests 全在这里）
├── model_eval/
├── storage/                  # provider 记忆落盘 + models/tools
├── data/{gaia,webwalkerqa,...}
└── assets/
```

## 4. 基础项目地图

### 4.1 语言与运行时
- Python 3；`requirements.txt` 依赖含 `sentence-transformers`、`scikit-learn`、`openai`、`python-dotenv`、`PyYAML` 等（provider 侧实际 import 见证据 E-011/E-012）。
- 无容器化/编排文件证据（顶层无 Dockerfile / compose）；运行依赖外部 OpenAI 兼容模型（`OPENAI_API_KEY` / `OPENAI_API_BASE` / `DEFAULT_MODEL`）。

### 4.2 入口
| 入口 | 作用 | 证据 |
|---|---|---|
| `evolve_cli.py` | 单阶段命令（analyze/generate/create/validate）与 `auto-evolve` | E-049 |
| `run_flash_searcher_mm_gaia.py` 等 | 数据集评测；`--memory_provider` 选择 provider；任务结束调用 `take_in_memory` | E-050 |
| `MemEvolve/core/auto_evolver.py` | 多轮锦标赛主循环 | E-046 |
| `FlashOAgents/agents.py` | 在 planning 与每个 step 注入 memory guidance | E-010 |

### 4.3 核心模块
| 模块 | 职责 | 关键事实 |
|---|---|---|
| `EvolveLab/base_memory.py` | 抽象基类 | 3 个抽象方法；无 update/delete/forget | E-001 |
| `EvolveLab/memory_types.py` | 类型与 dispatch | 13 枚举 + `PROVIDER_MAPPING`；`MemoryStatus` 仅 BEGIN/IN | E-002/E-003 |
| `EvolveLab/providers/*` | 13 套记忆实现 | 检索/写入/剪枝策略各异；阶段支持不对称 | E-007/E-008/E-009 |
| `MemEvolve/phases/*` | 分析/生成/创建/验证 | 代码生成 + 注释锚点改写 + 隔离验证 | E-035~E-044 |
| `MemEvolve/core/auto_evolver.py` | 多轮搜索与选择 | 每轮重置 storage、x 任务、top-t、finalists | E-046~E-048 |
| `MemEvolve/utils/run_provider.py` | 子进程调度 runner | 无超时；`concurrency=1` | E-049 |

### 4.4 核心数据结构
- `MemoryRequest{query, context, status, additional_params}`
- `MemoryItem{id, content, metadata, score?, type=TEXT}`
- `MemoryResponse{memories, memory_type, total_count, request_id}`
- `TrajectoryData{query, trajectory, result?, metadata?}`
- `MemoryStatus = {BEGIN, IN}`；`MemoryItemType = {TEXT, API}`
（证据 E-002~E-005，`EvolveLab/memory_types.py:11-105`）

### 4.5 状态与持久化
- 13 个 provider 中 **11 个以 JSON 文件**为默认后端（见 `json.dump`）；SkillWeaver 写 `.py`，Dynamic-Cheatsheet 写 `.txt`；无数据库/向量库依赖。
- `MemEvolve` 自身用 `state.json` 记录四阶段完成状态（E-035）。
- `AutoEvolver` 用 checkpoint JSON 支持中断续跑（E-048）。

### 4.6 测试体系
- **`EvolveLab/` 与 `MemEvolve/` 没有任何 `test_*.py` / `tests/`**；全仓测试仅存在于 vendored `mini-swe-agent/tests`（30 个文件）。
- 结论：核心元进化逻辑 **无自动化回归测试**，质量依赖运行时验证器 + 人工/论文实验。

### 4.7 配置
- `EvolveLab/config.py`：`DEFAULT_CONFIG["EvolveLab"]`（default_top_k=3、active_provider=agent_kb）+ 13 provider 段。
- `MemEvolve/config.py`：数据集/runner 映射 + 进化默认参数（x=20, t=2, y=5, m=3, seed=42, analysis_max_steps=20, creativity=0.5）。
- 运行参数主要由 CLI/`AutoEvolver` 覆盖，如 `num_systems`、`task_batch_x`、`top_t`。

### 4.8 权限与治理
- 无用户权限/多租户模型；治理集中在“谁能改代码”：`MemoryCreator` 直接改写 `EvolveLab/providers`、`memory_types.py`、`config.py`（E-039）。
- `PROVIDER_MAPPING` 是唯一 dispatch 权威（E-003）；validator 通过它动态 import（E-043）。

### 4.9 外部依赖
- LLM API（OpenAI 兼容）：分析、生成、摘要、判重、验证均调用模型。
- `sentence-transformers/all-MiniLM-L6-v2`（384 维）用于检索；部分 provider 用 TF-IDF 混合。
- 数据集：GAIA、WebWalkerQA(170 子集)、xBench DeepSearch、taskcraft、coin_flip。

## 5. 快照限制
- 浅克隆 → 无 git 历史、无 blame、无修复史；`failure-analyst` 的“失败与修复”只能从代码中的 fallback/异常分支推断，不能声称历史修复。
- 未安装依赖、未配置 API key、未下载数据 → 本轮无 S4/S5 运行证据。
