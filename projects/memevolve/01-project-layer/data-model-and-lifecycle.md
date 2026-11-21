# 01 Project Layer — Data Model & Lifecycle

## 1. 数据模型

### 1.1 请求 / 响应
```
MemoryRequest
├── query: str            # 任务问题
├── context: List[Message] # IN 阶段当前对话
├── status: MemoryStatus  # BEGIN | IN
└── additional_params: dict

MemoryResponse
├── memories: List[MemoryItem]
├── memory_type: MemoryType
├── total_count: int
└── request_id: str

MemoryItem
├── id: str
├── content: str
├── metadata: dict        # provider 自定义（score/usage_count/...）
├── score: float?
└── type: MemoryItemType  # TEXT | API
```
（E-006~E-008）

### 1.2 摄取信封
```
TrajectoryData
├── query: str
├── trajectory: List[Message]
├── result: Any?
└── metadata: dict
      ├── task_id
      ├── status: "success" | "error"
      ├── is_correct: bool
      └── full_query
```
（E-009；`run_flash_searcher_mm_gaia.py:321-331`）

**关键事实**：`metadata` 无 schema、无验证；正确性由调用方（LLM judge）写入，缺失时不同 provider 的解释不一致（E-038/E-058/E-082）。

## 2. 状态模型

| 状态载体 | 位置 | 生命周期 | 备注 |
|---|---|---|---|
| `MemoryStatus` | 请求级 | 单次调用 | 只有 BEGIN/IN（E-002） |
| 短期记忆 | provider 内存 + 任务结束落盘 | 单任务 | Lightweight `_task_snapshots` 以 query SHA256 为 key（E-044） |
| 长期记忆 | `storage/lightweight_memory` 等 JSON | 跨任务 | 默认不注入（E-042） |
| 提供缓存 | provider 内存 | 单任务 | Evolver `_last_provided_cache` 以 query 原文为 key（E-035/E-049） |
| 阶段状态 | `MemEvolve/.../state.json` | 单次进化 | 四阶段完成标记（E-055） |
| 锦标赛状态 | AutoEvolver checkpoint | 多轮 | 每步落盘可续跑（E-077） |

**没有统一的生命周期状态机**：删除、合并、失效只存在于个别 provider 的内部实现（Cerebra consolidation、Lightweight pruning、Evolver pruning），不是接口能力（E-001）。

## 3. 生命周期（记忆对象）

```
创建：take_in_memory(trajectory, metadata)
  ├── 成功门控（provider 各异）
  ├── LLM 摘要/抽取
  └── append / merge / revise
使用：provide_memory(request)
  ├── BEGIN：为初始计划提供全局经验
  └── IN：为当前步骤提供上下文（仅 5/13 支持）
强化：usage_count++（被再次检索/使用时）
衰减：prune（阈值/容量/LLM 选择）
消亡：从 JSON 删除 + 重建索引（无审计日志）
```

**可追溯性**：11/13 provider 用 `json.dump` 覆盖整个 JSON 文件；SkillWeaver 整文件写 `.py`，Dynamic-Cheatsheet 整文件写 `.txt`（E-090）；均无 append-only 日志、无版本号、无事务。唯一例外是 Agent-KB 的 append-only 四字段记录（E-026），但每次写入后重建索引。

## 4. 数据流（端到端）

```
数据集 jsonl
   ↓ runner
task query
   ↓
Agent loop ──BEGIN/IN──> Provider ──读──> storage/*.json
   ↓  trajectory                   ↑
   ↓                               │ 写入（成功后）
LLM judge → is_correct ────────────┘
   ↓
任务日志 JSON（accuracy/tokens/steps）
   ↓ trajectory_tools.aggregate
批次指标 → AutoEvolver 选择 → 下一代 provider
```

## 5. 数据契约的薄弱点（代码可见）

1. **无 schema 校验**：`TrajectoryData.metadata`、`MemoryItem.metadata` 都是任意 dict；provider 各自取键（E-009/E-038）。
2. **无迁移/版本**：JSON 结构变更没有 `version` 字段；升级 provider 后旧文件按新逻辑读取（E-090）。
3. **无写入幂等键**：除 Lightweight 的 SHA256 去重外，其余 provider 不做去重；同一轨迹重复摄取会重复写入（E-047）。
4. **无删除审计**：prune 直接从列表移除并重写文件，无法回溯“为什么删了这条记忆”（E-032/E-048）。
5. **API 类型记忆绕过存储语义**：Cerebra/SkillWeaver 返回可调用 tool，记忆的“内容”是代码对象而非文本，无法被统一序列化/审计（E-050/E-051）。

## 6. 生命周期中的并发问题（候选）

- runner 注释称“each thread creates independent instance”（E-080），但多个实例写同一 storage 文件时无文件锁；并发>1 时存在最后写入覆盖（候选 C-04，未被运行验证）。
- `run_provider` 固定 `--concurrency 1`（E-078），但 AutoEvolver 可并行多个 provider（`max_workers=3`，E-074），不同 provider 目录不同，冲突概率低；同一 provider 的多批次并行未见于代码。
