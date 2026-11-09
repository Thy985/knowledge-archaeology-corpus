# 00 · Overview · Hy-Memory 2.0

> Run `ARCH-2026-11-10-001` · `main@9737431cf8965124b115f0e26cb0ff1e5cafa323` · skill v3.2 · 2026-11-10
> 证据上限：**S3（Implemented）**。本轮未执行任何测试或真实运行；所有"运行行为"均来自代码路径推断，标注为 Observation。

## 1. 一句话定位

Hy-Memory 是一个**三场景、双系统**的 LLM Agent 记忆引擎：Chat 走 L0–L7 分层认知记忆，Working / Coding 走"会话事实 + 可复用经验"双层生产力记忆；Coding 额外挂一条**零 LLM 的代码版本演化链**。它把"记忆写入"从提示词工程推进到**可训练模型工件**（配套 30B-A3B MoE），并把产品层 MemHub 的职责严格限制在"管理面"。

## 2. 它真正值得被记住的三件事

### ① 写入策略成为模型工件，而不是 prompt 片段
- 仓库内 `hy_memory/agent/{extractor,reconciler,abstractor,intention_detector}.py` 是 System 1 的 LLM 抽取 / 冲突消解链路；配套模型 `Hy-Memory-30B-A3B` 在 HF 独立发布（README 声明）。
- 工程含义：**记忆质量的上限由"写入模型"决定**，而写入模型的训练目标（attributed RL）与检索层解耦。这使"换写入模型"成为一个可替换部件，而不是重写 prompt。
- 证据：README §Papers（文档声明，S2）；`agent/reconciler.py` 的 ADD/SUPERSEDE/UPDATE/DELETE 指令协议（S3）。

### ② 事实层与经验层分离：保底记忆 + 增值记忆
- `.md` / `docs/scene_coding_usage.md` 与 `README` 声称：每个 session 恰好 1 条 `WorkingFact`（无阈值），0~N 条 `Experience`（需过"价值阈值"）。
- 代码：`coding/store.py` 两张表 `working_fact` / `coding_memory_meta`，双向 id 链 `derived_experience_ids` ↔ `source_fact_ids`，`_crosslink` 取并集写回（`coding/writer.py:290-325`）。
- 工程含义：**"做过什么"不允许丢，"学到什么"允许没有**。这解决了纯经验型记忆的一个常见故障——一次没有复用价值的任务会让整轮记忆消失。

### ③ 版本链是"检索结果的附件"，不是第二个检索源
- `coding/chain_store.py` 用 SQLite 存 `thread/round`，零 LLM、零 embedding；`observe()` 在每次写入后 best-effort 挂链（`client.py:3030-3045`）。
- README 明确"a decorator, not a retriever"：向量层决定返回哪些 fact，链只把 fact 涉及代码的每一版（用户原话 + 真实 diff）附在下面。
- 工程含义：**摘要必然丢信息，但检索不该因此变成两条互相竞争的通道**。链补的是抽取规范主动砍掉的那类信息（"给坐标不要贴代码"），且只在需要时展开。

## 3. 三场景的边界（代码事实）

| scene | 路由条件 | 写入管线 | 检索层 |
|---|---|---|---|
| `chat` | 显式 `scene="chat"`，或 messages 无 tool（O(1) 短路），或 judge 判 chat | `MemoryWriter`（L1 raw → 可选 LLM 抽取 → reconcile） | L2/L3/L4 VDB + L0 隐藏 + L6 graph |
| `working` | 含 tool 且 judge 判 working；或显式 working + 含 tool | 与 coding **同一 pipeline / schema**，variant=working | productivity store |
| `coding` | 含 tool 且 judge 判 coding；或显式 coding + 含 tool | 同 working + 版本演化链 | productivity store + chain 装饰 |

**关键代码事实**：
- `_normalize_scene` 只接受 `chat/working/coding`，非法抛 `ValueError`（`client.py:861-890`）。
- `_scene_variant(None)` 返回 `working` 是**兼容 fallback**，真正 None 语义由三分类 judge 回填（`client.py:892-911`）。
- 强制 `coding` 但无 tool → 回退 chat（`client.py:2955-2965`）。
- judge 抛异常 → `is_coding=False`，静默走 chat（`client.py:2998-3005`）。

## 4. 与已考古项目的对照（Memory 四点正交）

| 项目 | 写入策略 | 记忆形态 | 版本/演化 | 治理 |
|---|---|---|---|---|
| `mem0` | 单遍 ADD-only / 抽取 | 扁平事实 + 向量 | 弱 | 项目级 |
| `graphiti` | 时序知识图谱 | 实体-关系-时间 | 强（时序边） | ABAC / 保留 / 审计 |
| `letta` | Agent 自编辑 | Core/Recall/Archival 三级 | DB undo/redo | executor read_only |
| **Hy-Memory** | **专训 30B 写入模型 + reconcile 协议** | **L0–L7 + 双层生产力 + diff 链** | **supersede 链 + 代码版本链** | **来源授权 + 用户/工作区隔离** |

**新增对照维度**：写入模型可替换性；"事实保底 / 经验增值"双层；版本链作为检索附件；跨应用共享（MemHub）。

## 5. 口径警示（必须与 Fact 层隔离）

README / 候选卡声明的厂商指标：
- LoCoMo 72.6%（1,527 道非 adversarial 题，开源框架第一）
- MemoryArena 55.31（+10.2 pt）
- CodeMemoryBench 64.2 / 35.6；SWE-ContextBench 28.3；存储占用 6.06%
- 腾讯自选股：多跳召回 50→90、冲突消解 67→83

**这些均未在本仓库代码中复现，且题目子集 / 裁判模型与 Mem0 的 92.5 不可横比。** 本轮只把它们记为"文档声明"，不进入 Engineering Knowledge 的因果结论。

## 6. 本轮覆盖与未覆盖

**已覆盖**：
- SDK 门面与 scene 路由、Chat 写入/读取主链路、reconcile 版本化、System 2 入口语义
- Working/Coding 双层结构与 crosslink、版本链三条 patch 来源与隔离键
- MemHub 管理面边界、来源授权、user/workspace 隔离、本地 API 防护
- 测试与 CI 配置、依赖与后端矩阵

**未覆盖**（显式声明）：
- `memhub/apps/desktop`、`memhub/apps/web` 的前端实现；`memhub/packages/*` 全量（568 py / 182k 行，产品层，与"记忆引擎"相关性低）
- `pipelines/system2_agent.py`（2,476 行）的 schema 归纳算法细节
- `pipelines/reader_base.py` 的全部 rerank / 图扩展分支
- 40+ `chore/*` build 分支
- 任何运行时行为（无测试执行、无真实调用）

## 7. 质量自评

| 项 | 值 |
|---|---|
| Facts（证据条目） | 36 |
| Engineering Knowledge | 29 |
| EK 边 | 58 |
| 平均出边 | 2.00 |
| 游离 EK | 0 |
| KO | 7（L3×5、L2×1、L4×1） |
| L5 | 0（不强行升维） |
| Candidates | 7 |
| Flow Edge | 47 |
| 精读代码 | ~1,150 行 / 322,396 行 ≈ 0.36%（聚焦主链路；未做全量通读） |
| 证据上限 | S3 |
| Independent Validation | CONFIRMED 6 / PARTIALLY 2 / MISSING 2（见 `06-validation/independent-validation.md`） |

> 本节数字为 **Reconciliation 后的最终口径**（见 `06-validation/reconciliation.md`），与 02/04 文件一致。

> 本轮是**主链路抽样考古**，不是全仓库覆盖。Coverage 的自评上限为"Memory 引擎主干 + MemHub 治理面"，不代表 `memhub` 产品层已被理解。
