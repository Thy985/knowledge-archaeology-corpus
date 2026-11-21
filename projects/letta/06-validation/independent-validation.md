# Independent Validation Report — ARCH-2026-11-09-001

> 审计角色：Independent Auditor（未参与 Package 写作）
> 方法：**先盲重建**。先独立重读 `repo/` 的关键实现（不打开 `package/`），形成 Independent Findings；随后才读取 Package 并对逐条比对。
> 约束：只读；不修改任何考古产物。

## 1. Independent Findings（盲重建结果）

| IF | 独立发现 | 证据 |
|---|---|---|
| IF-01 | 文件/行数：1156 文件、878 `.py`、248,417 行（排除 `.git`） | 独立 `os.walk` 重算，与 Package 数字逐位一致 |
| IF-02 | `core_tool_executor.py` 中 `read_only` 出现 **9 次**，均为写前检查 | `grep -c` + 逐行定位 320/336/354/530/665/691/744/899/959 |
| IF-03 | legacy `core_memory_append` / `core_memory_replace` 无 read_only、无 limit 校验 | `base.py:246-281` 全文 |
| IF-04 | legacy `rethink_memory` 在 block 不存在时直接 `set_block(new_block)` | `base.py:283-310` |
| IF-05 | `update_memory_if_changed_async` 以 `new_memory_str not in system_message.content[0].text` 判断变更 | `agent_manager.py:1762-1768` |
| IF-06 | **git-enabled 写入是 git-first，不是 DB-first**：`update_block_async` docstring "commits to git first"，步骤 1 git commit、步骤 2 sync PG | `block_manager_git.py:188-250` |
| IF-07 | `create_block_async` 同样 git-first（步骤 1 commit、步骤 2 sync） | `block_manager_git.py:288-330` |
| IF-08 | git→PG 同步调用**未传 `read_only` / `metadata`**（函数支持这两个参数） | `block_manager_git.py:232-244` vs `:82-92` |
| IF-09 | `sync_blocks_from_git` docstring：rebuild PG cache from git source of truth | `block_manager_git.py:564-568` |
| IF-10 | `insert_passage` 先 SQL（`create_agent_passage_async`），后 TPUF；异常 log，`strict_mode=True` 才 raise | `passage_manager.py:600-632` |
| IF-11 | `block_history.block_id` 外键 `ondelete=CASCADE` | `orm/block_history.py:42` |
| IF-12 | `_move_block_to_sequence` 在 **:760 与 :914 重复定义** | `grep -n "def _move_block_to_sequence"` |
| IF-13 | `BlockResponse.read_only` 标 `deprecated=True` | `schemas/block.py:103` |
| IF-14 | `Memory.compile()` 三分支：git / line-numbered / standard | `memory.py:688-745` |

## 2. 逐条比对判定

| # | Package 条目 | 判定 | 说明 |
|---|---|---|---|
| 1 | 规模数字（40 Facts 基础） | **CONFIRMED** | 独立重算逐位一致 |
| 2 | EK-09 `read_only` 执行点在 executor | **CONFIRMED** | 9 处精确匹配，无遗漏 |
| 3 | EK-10 legacy 无校验 | **CONFIRMED** | 全文确认；且 `rethink_memory` 亦无校验（Package 未强调） |
| 4 | EK-05 git-as-truth 注释 | **CONFIRMED** | docstring 原文一致 |
| 5 | EK-14/15 双写顺序与吞异常 | **CONFIRMED** | SQL 先、TPUF 后、strict 才 raise |
| 6 | EK-20 删除级联 | **CONFIRMED** | 外键 CASCADE |
| 7 | EK-01 compile 三分支 | **CONFIRMED** | 分支条件一致 |
| 8 | EK-19 read_only 是块级、deprecated | **CONFIRMED**（事实层） | 字段位置与 deprecated 标记均确认 |
| 9 | EK-03 blocks_agents 多对多 | **CONFIRMED** | 复合 FK + 唯一约束一致 |
| 10 | EK-08 字符串比较判变更 | **CONFIRMED** | 代码逐行一致 |
| 11 | **EK-06 / KO-02 "写路径仍先落 Postgres"** | **CONTRADICTED** | git-enabled 路径是 **git-first**（IF-06/07）。非 git 模式才是 DB 先写。Package 把两种模式混为一谈，方向说反 |
| 12 | KO-03 "影子路径"可触发 | **PARTIALLY_CONFIRMED** | legacy 实现确实存在且无校验；但**是否仍被注册给模型未证实**，Package 也标了 CAND-01——表述上仍用了"防护是否生效取决于路径"的强判断 |
| 13 | KO-04 共享 block 无法表达 per-agent 权限 | **OVER_GENERALIZED** | "字段不能表达"成立；但升到"共享与最小权限不可兼得"缺少实际越权证据，应降为 Observation |
| 14 | KO-01 "价值在准入成本而非存储介质" | **OVER_GENERALIZED**（轻度） | 三层结构与成本差异成立；"应独立设计"是规范性主张，超出 S3 证据 |
| 15 | EK-02 Archival 仅检索时进入上下文 | **PARTIALLY_CONFIRMED** | 未定位"检索结果注入 prompt"的具体代码路径，只确认了检索存在 |
| 16 | EK-16 Recall 压缩策略 | **PARTIALLY_CONFIRMED** | 两策略实现已读；默认启用哪一种未确认 |
| 17 | `_move_block_to_sequence` 重复定义 | **MISSING** | 生产方未写入任何 EK/Candidate，是潜在死代码/覆盖风险 |
| 18 | git→PG 同步漏 `read_only`/`metadata` | **MISSING** | 实现细节遗漏，可能造成缓存与真源字段不一致 |
| 19 | `letta/managers/` 22.9k 行未覆盖 | **MISSING**（覆盖缺口） | Package 已在边界声明中承认，但仍属未验证面 |
| 20 | `read_only` deprecated 的意图 | **NEEDS_HUMAN_REVIEW** | 需仓库历史或产品决策信息，代码本身无法判定 |

## 3. 判定统计

| 判定 | 数量 |
|---|---:|
| CONFIRMED | 10 |
| PARTIALLY_CONFIRMED | 3 |
| OVER_GENERALIZED | 2 |
| CONTRADICTED | 1 |
| MISSING | 3 |
| NEEDS_HUMAN_REVIEW | 1 |
| DOWNGRADED | 0 |
| **合计** | **20** |

## 4. 三个成功（经独立重读仍站得住的高价值结论）

1. **`read_only` 的执行点精确可数**：9 处全部位于 `core_tool_executor`，且都在写动作之前。这是"约束在执行层"最硬的证据。
2. **双写失败被吞是可复现的代码事实**：`except Exception` + `logger.error` + 仅 `strict_mode` 重抛，顺序为 SQL 先、TPUF 后。EK-15 成立。
3. **删除会级联删历史**：外键级联直接否定了"版本化可提供删除后审计"的假设，EK-20 成立。

## 5. 三个错误（必须修正）

1. **EK-06 / KO-02 的写入方向说反**：git-enabled 是 `commit to git` → `sync to postgres`（`block_manager_git.py:219-244`），不是"DB 先写"。应将结论改为："非 git 模式 DB 为热路径；git 模式 git 为真源、PG 为缓存；两模式均无跨存储事务。"
2. **KO-04 过度升维**：从"字段无法表达"推到"共享与最小权限不可兼得"，缺少实际越权/共享写扩散证据（CAND-06 未验证）。应降为 **Observation**，或改写为"当前模型不能表达 per-agent 差异"。
3. **KO-01 含规范性命中**："应独立设计"是建议不是事实。应保留结构描述，删除规范性措辞，或移入 Candidates。

## 6. 遗漏（Package 未记录但代码存在）

1. `block_manager.py` 的 `_move_block_to_sequence` **同名重复定义**（:760 / :914）。Python 后者覆盖前者；成因未核实，但这是可观察的实现风险。
2. `_sync_block_to_postgres` 支持 `read_only`/`metadata` 参数，但 `update_block_async` 调用时**未传**，导致 git 侧字段变更可能不回写 PG 缓存。
3. `rethink_memory` 在 block 不存在时**新建 block**（`set_block`），legacy 路径不仅能改还能建，权限面比 Package 描述更大。
4. `managers/` 22.9k 行未覆盖（Package 已声明边界，但审计仍将其列为覆盖缺口）。

## 7. Benchmark Case（供 skill 回归）

**Case: truth-source 迁移的方向判定**
- 输入：一个仓库同时存在"PostgreSQL 写入"与"git 写入"两条代码路径。
- 陷阱：只读 docstring（"git source of truth"）或只读 DB 侧代码，会得出"DB 是热路径、git 是备份"。
- 正确做法：逐路径读 `update/create/delete` 的执行顺序，区分 `git_enabled=True/False` 两个分支。
- 判定标准：必须能指出"git 分支：commit → sync PG；非 git 分支：直接 DB"，并说明无事务边界。
- 本轮的失败模式：Package 把两分支合并成一句"写路径仍先落 Postgres"，被独立审计判 CONTRADICTED。

## 8. 审计结论

Package 的事实层质量较高（10/14 关键事实 CONFIRMED，数字零偏差），但**因果/顺序类结论存在 1 处方向性错误**，且有 2 处过度升维。建议：

- 修正 EK-06、KO-02 的写入顺序表述；
- KO-04 降级为 Observation；
- 删除 KO-01 的规范性措辞；
- 补充 3 条遗漏为 EK 或 Candidate；
- 修正后再进入 Corpus 提交（Reconciliation 见 `06-validation/reconciliation.md`）。
