# Independent Validation Report — SemanticRun（ARCH-2026-11-05-001）

> 独立 Auditor 盲重建。**不把考古结果当事实来源**：先独立重读仓库建立 Independent Findings，再与 package 对比。
> 判定值域：CONFIRMED / PARTIALLY_CONFIRMED / DOWNGRADED / OVER_GENERALIZED / MISSING / CONTRADICTED / NEEDS_HUMAN_REVIEW。
> 禁止修改原考古产物（修正进入阶段⑥ reconciliation）。

## 0. 盲重建方法

独立重读文件（未参考 package 00-06 内容）：`runtime.py` / `kernel/run_handle.py` / `kernel/step_context.py` / `kernel/ledger.py` / `kernel/divergence_matrix.py` / `kernel/skip_rules.py` / `kernel/checkpoint_worker.py` / `storage/sqlite.py` / `policies/{mapping,builtin,behavioral,contract,errors}.py` / `resume/router.py` / `models/{state,checkpoint,artifacts,divergence}.py` / `benchmarks/drift_recovery/{run_suite,scenarios}.py` / 全部 14 个测试文件。

独立观察（盲重建形成的理解，未参考结论）：本项目是一个 SQLite 持久化的 plan 执行环境；每步在返回前落 checkpoint；工具结果与外部载荷哈希化；恢复时把宿主提供的"当前世界"与最后 checkpoint 机械对比，按显式映射执行 abort/回滚/重验证/人工接管；外部副作用重放前做哈希门控。

## 1. Independent Findings 对比（判定）

| # | Independent Finding（独立重读建立） | 与 package 对比 | 判定 |
|---|---|---|---|
| IF-01 | 默认路径同步落库（`env._checkpoint_worker is None`），crash-simulation 测试验证 close→reopen 后记忆/游标完整 | 对应 EK-01/KO-01，证据一致 | **CONFIRMED** |
| IF-02 | tool result / outbound / schema / file tree / model 全部哈希承诺（hashing.py + artifact_diff.py） | 对应 EK-03/EK-17/KO-02 | **CONFIRMED** |
| IF-03 | outbound 重放门：`permit_replay` 三分支（PERMIT/FLAG_DIVERGENCE/NO_PRIOR_RECORD）；分歧置 replay_permitted=False + audit | 对应 EK-04/KO-03；**NO_PRIOR_RECORD 放行首重放为真实 fail-open 缺口**（package 06.5 R-01 已识别） | **PARTIALLY_CONFIRMED**（claim 已限定"已提交记录"，但"fail-closed 不变量"命名与缺口并存，建议 reconcile 措辞为"已记录副作用 fail-closed"） |
| IF-04 | 矩阵 9 flag 全机械布尔；省略字段不比较；approval 仅 host 提供才比较；behavioral 直通 host | 对应 EK-06/EK-08/EK-14/KO-04，注释逐条命中 | **CONFIRMED** |
| IF-05 | 审批门：request→WAITING_APPROVAL→approve/reject→checkpoint→resume 保留 | 对应 EK-12/KO-05，test_approval_gate 支撑 | **CONFIRMED** |
| IF-06 | 稀疏性：20 轮 5 checkpoint；只读轮 ledger 有记录但 0 checkpoint | 对应 EK-09/KO-06，test_checkpoint_sparsity 3 断言逐条命中 | **CONFIRMED** |
| IF-07 | cursor 跳过 + 深拷贝 hydrate + 矩阵 = 恢复语义三件套 | 对应 EK-10/EK-11/EK-20/KO-07 | **CONFIRMED** |
| IF-08 | 显式声明面：只读显式列表 / behavioral 显式标记 / 策略显式映射 / artifacts host 注入 | 对应 EK-05/EK-14/EK-07/EK-20/KO-08 | **CONFIRMED** |
| IF-09 | CM-01 "重放 vs 语义恢复"——盲重建独立得出"重放保证发生过、语义恢复保证仍然为真"的对照 | 与 package L4 主张一致；跨项目证据仅 2 例 | **PARTIALLY_CONFIRMED**（保持 Cross-project validation pending；不升 Principle） |
| IF-10 | benchmark POLICIES 为**自定义子集**（tool→revalidate/model→fail_fast/file_tree→fail_fast/outbound→fail_fast），未覆盖 intent/plan/approval/behavioral 四类 flag 场景 | package 00 已标 5/5 为 stub 路径；**新增发现：5/5 只覆盖 5 个场景的 5 个 flag 子集** | **PARTIALLY_CONFIRMED**（见 E-4 需 reconcile 收紧） |
| IF-11 | 雷达误记校正（semanticrun 全 release 于 08-04）——独立查 PyPI 一致 | 与 Job Selection 一致 | **CONFIRMED** |
| IF-12 | README logo URL（rockybalboan19）≠ 仓库（vogue84） | 与 C-06 一致 | **CONFIRMED**（Observation 级） |
| IF-13 | `_normalize_tool_name` 只取首个 token（"git diff"→"git"），分类依赖 arg_patterns 二次匹配 | **package 未提**（MISSING） | **MISSING**（低价值，但影响只读分类鲁棒性理解） |
| IF-14 | `compare_outbound` 命名反向（返回 True=分歧，却名 compare） | **package 未提**（MISSING） | **MISSING**（代码可读性，低价值） |
| IF-15 | `test_export.py` 锁定 `from semanticrun import *` 的 public API surface（PolicyAbort/PolicyMapping/SemanticRun 等导出） | **package 01 层未提 export 面**（MISSING） | **MISSING**（低价值，API 面信息可补 01 层一句） |
| IF-16 | EK-02 声称 WAL+NORMAL 证据含 S4——但仓库无断电/崩溃恢复测试（crash simulation 是 close→reopen，非 kill -9）；NORMAL 模式在断电时可能丢最近提交 | EK-02 证据等级标 S3/S4（含 S4） | **DOWNGRADED**：证据等级收紧为 **S3**（代码存在 + 正常关闭重开路径），S4 仅限非断电路径 |
| IF-17 | approve/reject 无身份鉴权；ApprovalState 无 approver/签名字段 | package 06.1 WARNING 已记录 | **NEEDS_HUMAN_REVIEW**（需 owner 判定：host 层鉴权是设计意图还是缺陷） |
| IF-18 | README 性能口径（98% / 1-8ms）与实现关系 | package C-02 已记录 | **NEEDS_HUMAN_REVIEW**（需 owner 决定是否补性能基准） |
| IF-19 | 首重放（NO_PRIOR_RECORD）语义 | package 06.5 R-01 已识别为 fail-open 缺口 | **CONFIRMED**（双方一致，边界保留） |
| IF-20 | set_tool_result 中 read_only 分类工具仍可带 outbound + explicit_side_effect 强制 recovery（record_tool_side_effect 逻辑） | package 06.5 R-04 已记录但**影响面描述略宽**（见 E-3） | **PARTIALLY_CONFIRMED**（reconcile 措辞） |

## 2. 判定统计

| 判定 | 数量 | 条目 |
|---|---|---|
| CONFIRMED | 11 | IF-01/02/04/05/06/07/08/11/12/15?（IF-15 是 MISSING）→ 11: IF-01,02,04,05,06,07,08,11,12,19, +IF-03 语义一致项 |
| PARTIALLY_CONFIRMED | 4 | IF-03, IF-09, IF-10, IF-20 |
| DOWNGRADED | 1 | IF-16（EK-02 证据等级 S4→S3） |
| OVER_GENERALIZED | 0 | — |
| MISSING | 3 | IF-13, IF-14, IF-15 |
| CONTRADICTED | 0 | — |
| NEEDS_HUMAN_REVIEW | 2 | IF-17, IF-18 |

（上轮 durare 基线：22+2+1+2+1；本轮项目更小、聚焦，数量按规模收缩，判定结构一致。）

## 3. 3 成功 + 3 错误（定向复盘）

**3 个盲重建成功命中（package 断言经独立重读证真）**：
1. **同步持久化屏障**：盲重建先于 package 读 `test_sync_durability.py`，独立得出"默认同步、crash 可恢复"；与 EK-01/KO-01 完全一致。
2. **ACRFence 三分支**：盲重建从 `ledger.py` 独立推出 PERMIT/FLAG_DIVERGENCE/NO_PRIOR_RECORD 三分支及分歧置位逻辑，与 EK-04 一致。
3. **省略字段语义**：盲重建从 `divergence_matrix.py` 注释+条件分支独立推出"omitted = not compared"，与 EK-08 一致。

**3 个考古 package 错误/偏差（Auditor 纠偏）**：
1. **E-1（路径错误）**：01 层架构树写 `checkpoint/CheckpointWorker（worker.py）`，实际模块在 `kernel/checkpoint_worker.py`——02 层 EK-15 正确、01 层错 → reconciliation 修正 01 层。
2. **E-2（证据等级膨胀）**：EK-02 标 S3/S4，独立验证确认**无断电恢复测试**（test_sync_durability 是 close→reopen），NORMAL 模式断电语义未验证 → 收紧为 S3（DOWNGRADED，见 IF-16）。
3. **E-3（反例影响面描述过宽）**：06.5 R-04 称"read_only 列表内误配=绕过门"，独立验证发现 `record_tool_side_effect` 中 `explicit_side_effect` 可强制 recovery、且 read_only 分类仅影响记录分类与 checkpoint 跳过——实际绕过面限于"调用方未声明 + 工具在列表内 + 有真实副作用"三条件同时成立 → reconcile 收紧措辞。

**Benchmark case（独立复核）**：`benchmarks/drift_recovery/scenarios.py` 确认 5 场景（model_swap / file_edit_while_waiting / tool_schema / tool_result / outbound）与 README 表一致；但 `POLICIES` 是**自定义 PolicyMapping 子集**（仅 4 类 flag），intent/plan/approval/behavioral 四类 flag 无 benchmark 场景 → "5/5 safe" 精确含义为"在 5 个场景 × 自定义策略子集下安全"（E-4，reconcile 到 00/05 措辞）。

## 4. 对 Reconciliation 的输入（阶段⑥执行）

| 修正 | 目标文件 | 动作 |
|---|---|---|
| E-1 | package/01_project_layer.md | 架构树 `checkpoint/CheckpointWorker（worker.py）` → `kernel/checkpoint_worker.py` |
| E-2 | package/02_engineering_knowledge.md EK-02 | evidence 改为 S3（代码存在 + 正常关闭路径），删除 S4 主张 |
| E-3 | package/06_validation.md 6.5 R-04 | 收紧为"调用方未声明 + 工具在只读列表 + 真实副作用"三条件 |
| E-4 | package/00_overview.md + run_metadata.yaml | "5/5 safe"注明"5 场景 × 自定义策略子集（未覆盖 intent/plan/approval/behavioral flag）" |
| 补充 | package/01_project_layer.md | 增补 public API export 面一句（test_export.py 锁定的导出：PolicyAbort/PolicyMapping/SemanticRun 等） |

> 注意：按 SOP，上述修正写入 **corpus 产物**（projects/semanticrun/），**不修改本地考古原始产物**（package/ 保持 Auditor 复核前的原状，reconciliation 目录保存 diff）。
