# 01 Project Layer — Architecture（MemEvolve / EvolveLab）

## 1. 一句话定位
MemEvolve 是一个**记忆系统的元进化框架**：它不让研究者手写“更好的记忆模块”，而是把已有记忆实现（EvolveLab 的 13 个 provider）当作基因库，用 LLM 分析失败轨迹 → 生成新的 provider 代码 → 写入注册表 → 隔离验证 → 在真实任务上与基座做锦标赛，胜者进入下一轮。

## 2. 分层架构（实际代码边界）

```
┌──────────────────────────────────────────────────────────────┐
│ 评测/运行层  run_flash_searcher_*.py  /  evolve_cli.py        │
│   - 数据集 runner，按 --memory_provider 选择实现              │
│   - 任务结束后调用 take_in_memory                             │
└───────────────┬──────────────────────────────────────────────┘
                │ TrajectoryData
┌───────────────▼──────────────────────────────────────────────┐
│ Agent 运行时  FlashOAgents/agents.py                         │
│   - planning 前注入 BEGIN memory（user 消息）                 │
│   - 每个 step 注入 IN memory（user 消息，带 context）         │
└───────────────┬──────────────────────────────────────────────┘
                │ MemoryRequest / MemoryResponse
┌───────────────▼──────────────────────────────────────────────┐
│ 记忆抽象层  EvolveLab/                                       │
│   base_memory.py  3 方法抽象                                  │
│   memory_types.py 13 枚举 + PROVIDER_MAPPING（唯一 dispatch） │
│   providers/      13 套实现（JSON 存储 + 嵌入检索 + LLM 摘要）│
└───────────────▲──────────────────────────────────────────────┘
                │ 直接读写源码/配置（非 API）
┌───────────────┴──────────────────────────────────────────────┐
│ 元进化层  MemEvolve/                                          │
│   phases/  analyze → generate → create → validate             │
│   core/    memory_evolver（单轮）/ auto_evolver（多轮锦标赛）  │
│   utils/   run_provider（子进程）/ trajectory_tools（指标）    │
│   validators/ SWEAgentValidator（自动修复）                   │
└──────────────────────────────────────────────────────────────┘
```

**架构判据**：元进化层不是通过插件 API 注册新 provider，而是**直接修改 `EvolveLab` 源码与配置**（E-063）。因此“进化层”与“被进化层”之间没有进程/接口隔离，只有验证阶段的目录隔离（E-069/E-070）。

## 3. 核心抽象

| 抽象 | 语义 | 约束 |
|---|---|---|
| `BaseMemoryProvider` | 记忆实现必须实现 `provide_memory` / `take_in_memory` / `initialize` | 无 update/delete/merge/forget 一等操作（E-001） |
| `MemoryRequest.status` | 仅 BEGIN / IN 两相 | 无 END / FAILED / TASK_END（E-002） |
| `MemoryItem.type` | TEXT（注入 prompt）或 API（可调用工具） | 二元载荷（E-005） |
| `PROVIDER_MAPPING` | `MemoryType -> (class, module)` | 新增系统必须同时改 enum + mapping（E-004） |
| `TrajectoryData.metadata` | 正确性、task_id、status 等 | 无 schema 校验（E-009/E-081） |

## 4. 控制流（单任务）

1. runner 解析 `--memory_provider`，通过 `MemoryType(value)` + `PROVIDER_MAPPING` 动态 import（E-004/E-078）。
2. `provider.initialize()` 建立 JSON 存储与索引。
3. `ToolCallingAgent` 在 planning 前调用 `provide_memory(BEGIN)`，把返回文本包成 user 消息插入（E-084）。
4. 每个 step 调用 `provide_memory(IN)`，带上当前 context 与 step_number（E-086）。
5. 任务结束（成功或异常）后构造 `TrajectoryData`，调用 `take_in_memory`；正确性由 LLM judge 给出（E-081/E-082）。

## 5. 控制流（元进化单轮）

```
基座 provider 跑 x 任务
        ↓
AnalysisAgent（20 步，工具读轨迹/记忆库）
        ↓  report
Generator（读 provider 源码 + memory_types，temperature=0.3+0.9c）
        ↓  {provider_code, config_updates, enum_info}
Creator（写 provider.py → 改 enum/mapping → 改 config.py）
        ↓
Validator（静态子串检查 → 隔离环境 import/initialize/BEGIN/take_in）
        ↓  失败则 SWEAgentValidator 自动修复 → 同步回真实目录
        ↓  validated list
与基座在同一 batch 并行评测 → top-t → finalists 复评 → winner
```

## 6. 关键设计取舍（代码可见）

| 取舍 | 实现 | 代价 |
|---|---|---|
| 用“改源码”代替“插件注册” | Creator 直接写 `EvolveLab/providers/*.py`、`memory_types.py`、`config.py`（E-063） | 无事务、无回滚；失败会留下半成品（E-064） |
| 用“隔离副本 + 清模块缓存”代替容器 | `shutil.copytree` + `sys.path` + `del sys.modules`（E-069/E-070） | 依赖同进程全局状态；清理只在通过时发生 |
| 用“LLM 自动修复”代替人工返工 | `SWEAgentValidator.fix_memory_system`（E-068） | 验证器同时成为修复者，独立性受损 |
| 用“每轮清空 storage”保证公平 | Step0 重命名为时间戳备份（E-071） | 跨轮记忆积累被丢弃；备份目录无清理策略 |
| 用“固定小 batch + 重采样”控制成本 | x=20, t=2, y=5, m=3（E-089） | 选择方差大；无置信区间/显著性检验 |

## 7. 外部依赖与运行前提

- OpenAI 兼容 LLM：分析、生成、摘要、判重、judge、自动修复全部依赖（E-056/E-061/E-082）。
- `sentence-transformers/all-MiniLM-L6-v2`（384 维）与 scikit-learn TF-IDF（E-011~E-013）。
- 数据集与 runner 映射（E-088）；runner 以子进程运行且无超时（E-078）。

## 8. 架构风险摘要（详见 05/06）
1. **验证器-修复者耦合**：验证失败会被自动修复并回写真实目录（E-067/E-068）。
2. **非原子代码变异**：Creator 的多步写入无回滚（E-064）。
3. **阶段不对称**：7/13 provider 只在 BEGIN 提供记忆，IN 阶段实际无记忆（E-021）。
4. **成功判定漂移**：至少 4 套实现，且 MemP/Evolver 缺失元数据时默认成功（E-038/E-058/E-059）。
