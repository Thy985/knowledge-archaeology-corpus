# Candidates（6 条未验证假说）

> 严格区分：以下**不是知识**，是假说。禁止被下游当作 Fact/EK 引用。

## CAND-01 · legacy memory 工具是否仍可被模型调用
- **Hypothesis**: `core_memory_append` / `core_memory_replace` / `rethink_memory` 仍通过某个 tool set 注册给部分 agent 类型（如 memgpt_v2_agent / sleeptime_agent），因此"legacy 无 read_only 校验"是可触发的真实旁路，而不只是遗留代码。
- **现有证据**: `base.py:246/263/283` 有完整实现（非 `NotImplementedError`）；`Memory.compile()` 仍为 `memgpt_v2_agent` / `letta_v1_agent` 保留专用渲染分支（`memory.py:692-703`）。
- **缺失证据**: 工具注册表 / agent 类型 → function set 的映射；未检索 `tool_rules` 默认配置。
- **验证路径**: 运行 `pytest tests/test_memory.py`；或在测试库创建 `memgpt_v2_agent`，检索其 tool schema 是否含 `core_memory_append`。
- **若成立的影响**: KO-03 从"理论旁路"升级为"实际旁路"。

## CAND-02 · git 与 Postgres 并发写会静默分叉
- **Hypothesis**: 同一 agent 的两个并发请求同时改 block 时，Postgres 有 `version` 乐观锁，git 侧按 commit 顺序覆盖；失败方可能只在 DB 报错而 git 已写入（或反之），最终以 `sync_blocks_from_git` 单向覆盖，丢失一侧修改。
- **现有证据**: `Block.version` 乐观锁（`orm/block.py:56-60`）；`sync_blocks_from_git` 单向 DB←git（`block_manager_git.py:564`）；无冲突合并代码。
- **缺失证据**: `update_block_async` 在 git 模式下的写入顺序与异常处理（`block_manager_git.py:188-288` 未逐行读完）。
- **验证路径**: 双并发写同一 label 的集成测试；检查 git commit 与 DB 更新的事务边界。
- **若成立的影响**: KO-02 需要补一条"并发下的真源选择"约束。

## CAND-03 · Turbopuffer 双写造成检索视图不一致
- **Hypothesis**: 非 strict 模式下 TPUF 写失败后，SQL 有 passage、向量库无对应条目；同一 archive 的检索会漏掉这些 passage，且无对账任务修复。
- **现有证据**: `passage_manager.py:620-628` 捕获异常仅 log。
- **缺失证据**: 是否存在离线 reconcile job；`strict_mode` 在哪些调用点为 True。
- **验证路径**: 检索 `strict_mode=True` 的调用点；查 jobs router / cron 是否有 TPUF 对账。
- **若成立的影响**: EK-15 从"失败路径"升级为"已知一致性缺口"。

## CAND-04 · `read_only` 标记 deprecated 意味着权限模型迁移中
- **Hypothesis**: `BlockResponse.read_only` 被标 deprecated，但 ORM 与 executor 仍依赖它，说明团队计划把权限迁到别处（如 agent 级策略或路径模型），当前处于双轨期。
- **现有证据**: `schemas/block.py:103`（deprecated）+ `orm/block.py:48`（仍生效）+ executor 9 处检查。
- **缺失证据**: 迁移计划文档 / PR；deprecated 字段的实际弃用时间线。
- **验证路径**: 需仓库历史（本轮浅克隆无历史）或 letta-code 的权限模型对照。
- **若成立的影响**: KO-04 的"缺口"可能是"过渡态"，而非终态设计。

## CAND-05 · 生产环境实际启用哪种 summarizer 策略
- **Hypothesis**: 代码支持 compact / self / sliding_window / all 四种，但默认配置只启用其一；不同策略对"记忆保真"的影响差异大。
- **现有证据**: `summarizer/` 四实现；`summarizer_config.py` 有 `CompactionSettings` 默认值（未逐行读完）。
- **缺失证据**: 默认 settings 的实际取值；服务端创建 agent 时是否覆盖。
- **验证路径**: 读 `summarizer_config.py` 全文 + `agent_manager` 创建路径的默认参数。
- **若成立的影响**: KO-01 的"Recall 压缩"描述需要具体到策略。

## CAND-06 · 共享 block 的写入会扩散到所有宿主
- **Hypothesis**: 若 block X 同时挂 agent A 与 agent B，A 的一次 `memory_replace` 会改同一个 `block` 行，B 下次 compile 立即看到新值；没有"写时复制"或宿主级隔离。
- **现有证据**: `blocks_agents` 多对多 + `update_block_async` 按 `block_id` 更新单行（`block_manager.py:211`）。
- **缺失证据**: 是否存在 conversation-isolated block（`update_memory_if_changed_async` 注释提到 "conversation-isolated blocks with different IDs"，`agent_manager.py:1776-1777`）。
- **验证路径**: 读 `blocks_conversations` 的创建逻辑；构造双 agent 共享 block 的测试。
- **若成立的影响**: 直接强化 KO-04，并可作为跨 agent 记忆污染的真实案例。
