# Independent Validation Report — ARCH-2026-11-13-001

> 角色：独立 Auditor。流程：先**不读**考古产物，按 repository-mapper → code-analyst 的顺序独立重读仓库，形成 Independent Findings；再与 `package/` 对比。
> 只读；未修改任何考古产物。
> 仓库：`/home/user/.doubao/agent_mode/workspace/archaeology-jobs/ARCH-2026-11-13-001/repo`，commit `6035d56`。

## 1. Independent Findings（盲重建摘要）

| IF | 独立发现 | 证据 |
|---|---|---|
| IF-01 | 记忆接口只有 3 个抽象方法，无 update/delete/forget | `base_memory.py:10-51` |
| IF-02 | 状态枚举只有 BEGIN/IN，注释 “only these two status” | `memory_types.py:11-15` |
| IF-03 | 13 个 MemoryType 与 13 项 PROVIDER_MAPPING 一一对应 | `memory_types.py:18-63` |
| IF-04 | BEGIN 注入发生在 planning 前，IN 注入发生在每个 step，且都是 USER role | `agents.py:413-425`、`782-795` |
| IF-05 | 7 个 provider 的 `MemoryStatus` 分支只处理 BEGIN；1 个只处理 IN；5 个双处理 | 13 个 provider 文件 grep |
| IF-06 | Lightweight 长期注入默认 False，类与 config 两处一致 | `lightweight:108-116`、`config.py:44-53` |
| IF-07 | Evolver `_save_store` 不写 embeddings，initialize 长度不符时全量重算 | `evolver:161-175`、`123-159` |
| IF-08 | MemP 在 `is_correct` 缺失时默认 `"true"` | `memp:480` |
| IF-09 | `PhaseValidator` docstring 说 “without automatic fixes”，构造函数默认 `enable_auto_fix=True`，并会 sync 回真实目录 | `phase_validator.py:29-43`、`197-214`、`814-835` |
| IF-10 | Creator 写入顺序 provider → memory_types → config；config 校验失败时前两步已落盘 | `memory_creator.py:86-284` |
| IF-11 | AutoEvolver 每轮把 storage 重命名为时间戳备份，provider 从空开始 | `auto_evolver.py:812-840` |
| IF-12 | `run_provider` 用 Popen.wait()，无 timeout，concurrency=1 | `run_provider.py:48-86` |
| IF-13 | `EvolveLab`/`MemEvolve` 无测试；测试只在 vendored mini-swe-agent | `find` 全仓 |
| IF-14 | 持久化并非全是 JSON：SkillWeaver 写 `.py` 技能文件；Dynamic-Cheatsheet 写 `.txt` | `skillweaver:41`；`dynamic:80-87` |
| IF-15 | 摄取失败路径也被调用：异常任务以 `is_correct=False` 调 `take_in_memory` | `run_..._gaia.py:369-388` |
| IF-16 | runner 对 `take_in_memory` 失败只记 warning，无重试/补偿 | `run_..._gaia.py:336-338`、`386-388` |
| IF-17 | Evolver 的 `_last_provided_cache` 是实例内存态；provider 每线程实例化，跨任务/跨线程不共享 | `evolver:388-410`；`run_..._gaia.py:594-607` |

## 2. 对比结果（判定表）

| # | 考古包断言 | 独立判定 | 依据/说明 |
|---|---|---|---|
| 1 | 3 方法窄接口 | **CONFIRMED** | IF-01 |
| 2 | 只有 BEGIN/IN | **CONFIRMED** | IF-02 |
| 3 | 13 providers / mapping | **CONFIRMED** | IF-03 |
| 4 | 记忆以 user 消息注入 planning 与每步 | **CONFIRMED** | IF-04 |
| 5 | 7/1/5 阶段矩阵 | **PARTIALLY_CONFIRMED** | IF-05 证明“只写了 BEGIN 分支”，但未逐行确认所有 IN 返回空响应对象；判“能力缺失”成立，“空返回”是推断 |
| 6 | Lightweight 长期默认关闭 | **CONFIRMED** | IF-06 |
| 7 | Evolver 存盘/装载不对称 | **CONFIRMED** | IF-07 |
| 8 | MemP 缺失默认成功 | **CONFIRMED** | IF-08 |
| 9 | Validator 文档称不自动修复、实现默认开启 | **CONTRADICTED（对文档）** | IF-09：文档声明与实现相反；作为“实现事实”成立，作为“文档承诺”被推翻 |
| 10 | Creator 非原子、无回滚 | **PARTIALLY_CONFIRMED** | IF-10 证明顺序与失败点；“无回滚”由代码中不存在 rollback 推断，未运行复现 |
| 11 | 每轮 storage 重置 | **PARTIALLY_CONFIRMED** | IF-11 仅核到行号区间（浅克隆无历史，未观察运行） |
| 12 | Runner 无 timeout | **CONFIRMED** | IF-12 |
| 13 | 核心模块无测试 | **CONFIRMED** | IF-13 |
| 14 | “13/13 以 JSON 持久化” | **DOWNGRADED（scope 错误）** | IF-14：至少 2 个例外（.py / .txt）；应改为“11/13 JSON 文件，1 个 .py，1 个 .txt” |
| 15 | “所有写入都是 json.dump 覆盖整文件” | **DOWNGRADED** | 同上；SkillWeaver 写 Python 源码，Dynamic 写文本 |
| 16 | KO-06 为 L4 模型 | **OVER_GENERALIZED → DOWNGRADED** | 单项目 7 条 EK，无第二项目；应降 L3 Pattern 或保留 L4 但明确 “Hypothesis” |
| 17 | KO-07 为 L4 模型 | **OVER_GENERALIZED → DOWNGRADED** | 同上；且 auto-fix 的“通过率”未测量 |
| 18 | “13 = 12 baseline + Evolver”（Day5 扫描口径） | **NEEDS_HUMAN_REVIEW** | README 说 “11 baselines + 2 evolved”；代码无 evolved 标记；两种口径冲突，需论文附录裁决 |
| 19 | “记忆差异主要在写入形态”（KO-01） | **PARTIALLY_CONFIRMED** | 5 种写入形态可核；但 Cerebra consolidate 同时改结构与权重，MobileE 不用 embedding，概括范围需收紧 |
| 20 | 摄取失败只影响单任务 | **MISSING（新发现）** | IF-15/16：失败轨迹会进入记忆且写失败只 warning，可能污染记忆，考古包未单列 |
| 21 | Evolver 使用量反馈可跨任务生效 | **MISSING（新发现）** | IF-17：缓存是实例态，每线程新实例，跨任务不保证；考古包未提 |

## 3. 判定统计

| 判定 | 数量 |
|---|---|
| CONFIRMED | 8 |
| PARTIALLY_CONFIRMED | 4 |
| DOWNGRADED | 3（1 scope + 2 abstraction） |
| OVER_GENERALIZED | 2（计入 DOWNGRADED 的抽象两项） |
| CONTRADICTED | 1（文档 vs 实现） |
| NEEDS_HUMAN_REVIEW | 1（baseline 口径） |
| MISSING | 2（失败轨迹摄取、Evolver 实例态缓存） |
| **合计** | **21 项检查** |

## 4. 三个成功（经独立重读仍成立）

1. **接口窄化是结构约束**（IF-01）：13 个实现确实都只能通过 3 个方法表达生命周期，KO-01 的底座成立。
2. **注入机制是提示词层而非工具层**（IF-04）：planning 与每步都以 USER 消息注入，且包裹标记固定；这解释了为什么 API 型记忆是唯一例外。
3. **验证器-修复者耦合**（IF-09）：文档与实现相反，且真实目录会被覆盖；KO-07 的机制链（EK-42→43→45）无断点。

## 5. 三个错误（必须修正）

1. **“全部 JSON 持久化”错误**（IF-14）：SkillWeaver `.py`、Dynamic-Cheatsheet `.txt`。必须在 Corpus 中改为“11/13 JSON + 1 py + 1 txt”，并同步修正 E-090、Snapshot 4.5、Data Model 3。
2. **“13 = 12 baseline + Evolver”口径冲突**（IF-18）：README 是 “11 + 2”。不得在 Corpus 中保留单一说法；应并列两套口径并标 NEEDS_HUMAN_REVIEW。
3. **KO-06/KO-07 升到 L4 过快**（IF-16/17）：单项目证据不足以支撑“模型”；应降为 L3 Pattern（或 L4 但 Epistemic=Hypothesis）。否则违反“单案例→Pattern、项目经验→通用 Principle”禁令。

## 6. 遗漏（考古包未覆盖，独立发现）

- **O-1 失败轨迹摄取**：异常分支也 `take_in_memory`，而多数 provider 的失败处理不一致（MemP 会 revise，Agent-KB 会丢弃）。这会让“失败经验”以不同方式进入不同系统，影响跨系统比较。
- **O-2 Evolver 实例态缓存**：`_last_provided_cache` 不持久化，provider 每线程实例化；使用量/成功率可能永远不更新，导致打分公式失效。
- **O-3 摄取失败静默**：`take_in_memory` 返回 False 只 warning，无重试/死信；长期运行会丢记忆且无监控指标。
- **O-4 清理缺口**：隔离目录只在 passed 时清理（`phase_validator.py:804-810`），失败候选会累积磁盘。
- **O-5 文档-代码的双重事实源**：README 与 config 默认值冲突（Evolver prune 0.2 vs 0.3），考古包只在 EK-32 提及，未列入 conflict list。

## 7. Benchmark Case（供 Skill CI 使用）

**Case ID**: `memevolve-validator-independence`

- **输入**：一个故意违反契约的 provider 候选（`provide_memory` 返回 `None`、`initialize` 返回 False），以及 `enable_auto_fix=True/False` 两种配置。
- **期望**：
  1. `enable_auto_fix=False` → verdict=failed；
  2. `enable_auto_fix=True` → 若修复后通过，记录 `fixed=true` 且不得把“通过”计为“原始生成质量”；
  3. 真实目录在未通过前不得被修改。
- **考察点**：Validation 阶段是否能区分“原始通过”与“修复后通过”；是否会把修复结果回写真实环境；报告是否保留修复痕迹。
- **当前代码预期**：1 成立；2 部分成立（有 sync 但无 fixes 统计）；3 成立（只在 passed 后 sync）。若未来实现变化，此 case 可作回归锚。

## 8. 结论

`PARTIALLY_VALIDATED`：
- 机制层事实（接口、注入、阶段、写入形态、验证链）经独立重读成立；
- 但存在 3 处必须修正的错误（持久化口径、baseline 口径、KO 层级）与 5 处遗漏；
- 无运行证据（S4/S5），所有性能/有效性结论不可用；
- 建议 Corpus 合并时执行 Reconciliation 的 R-01~R-05（见 `package/06_validation/reconciliation.md`）。
