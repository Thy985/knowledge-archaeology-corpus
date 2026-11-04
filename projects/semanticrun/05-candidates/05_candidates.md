# 05 Candidates — SemanticRun（ARCH-2026-11-05-001）

> 未验证假说。严格区分 Hypothesis / 暂定模式 / 跨项目 Candidate；不因"结论漂亮"升层。

## C-01 checkpoint sparsity 的误分类风险（Hypothesis）
- **claim**：checkpoint 稀疏性依赖 `classify_action` 正确分类只读动作；若某"名义只读"工具实际有副作用（或显式列表漏配），则该轮**不 checkpoint**，崩溃后丢失恢复点——稀疏与安全之间存在未被本项目测试覆盖的边界。
- **当前证据**：EK-05/EK-09（S4）；`tests/test_checkpoint_sparsity.py` 只覆盖显式列出的只读工具
- **缺失证据**：无"分类错误→恢复点丢失"的失败场景测试；无对非标准工具名的鲁棒性测试（`_normalize_tool_name` 只取首个 token）
- **验证路径**：构造"显式列表外的实际有副作用工具"场景，断言 checkpoint 行为；或对 READ_ONLY_PATTERNS 做属性测试

## C-02 性能声明无仓库内证据（Observation → 待验证）
- **claim**：README 声明 "checkpoint 1-8ms / memory <4MB / token savings up to 98%"，但仓库内 benchmark（drift_recovery）**只测安全（drift 检测）不测性能**；无任何性能基准或 profiling。
- **当前证据**：README 文本（S0/S3 声明级）；`benchmarks/drift_recovery/results.json` 为安全场景结果
- **缺失证据**：性能基准、规模测试（万级 tool result / 大 file tree 的 hash 耗时）、真实 agent 负载的 token savings 对照
- **验证路径**：对 hash_tool_result / checkpoint 序列化 / file tree hash 做基准；对比 naive/盲恢复的 token 成本

## C-03 SQLite 单写者并发竞态未处理（Hypothesis）
- **claim**：SQLiteStorage 无显式锁/连接池管理；`CheckpointWorker` 后台线程与主线程并发写同一连接；多进程同时 resume 同一 run 的竞态未定义。
- **当前证据**：`storage/sqlite.py` 无 threading.local / 锁；`checkpoint_worker.py` 线程写同一 `self._conn`（S3 代码观察）
- **缺失证据**：无并发测试（多线程/多进程）；无 SQLite "database is locked" 处理
- **验证路径**：并发 checkpoint 压力测试；多进程 resume 同一 run_id

## C-04 drift-recovery 5/5 的 S4→S5 缺口（Observation）
- **claim**：benchmark 默认确定性 stub（无 OPENROUTER_API_KEY 时）；真实模型路径（revalidate 提示生成、行为漂移判定）未在本环境验证——5/5 是"机制正确性"证据（S4），不是"真实模型环境安全"证据（S5）。
- **当前证据**：`benchmarks/drift_recovery/run_suite.py` 分支逻辑；44 tests 全过（stub 路径）
- **缺失证据**：真实模型（OpenRouter）下 5 场景全通过；revalidate 模板在真实 LLM 输出下的质量
- **验证路径**：设置 OPENROUTER_API_KEY 重跑 run_suite；记录真实模型路径结果

## C-05 跨项目假说：artifact-hash gate 可迁移到 Tafcm 审计面（Cross-project Candidate）
- **claim**：SemanticRun 的"提交物哈希承诺 + 恢复时机械对比 + fail-closed 重放门"模式可迁移到 Tafcm 的 ADI/审计面——对 agent 每次影响状态的动作做哈希承诺，复查时对比，不一致即拒（与 durare 的 verify-before-act 同构）。
- **当前证据**：本项目 S4；durare（11-04）同类机制 S4/S5；Tafcm 的 AGENTS.md 权限矩阵 + repair-verify read-only 重新证明（既有）
- **缺失证据**：Tafcm 侧无 artifact hash 承诺实现；未做过跨项目实验
- **验证路径**：在 Tafcm 审计器上做概念验证（record → verify → replay gate）；若成立可升 L4

## C-06 README logo 与仓库名不一致——改名/迁移史证据不足（Hypothesis）
- **claim**：README logo URL 指向 `rockybalboan19/SemanticRun`，而仓库实际为 `vogue84/SemanticRun`；PyPI 描述（ysharmcode/SemaRun）与代码 API（SemanticRun/PolicyMapping）不同——疑似账号/仓库改名迁移史，但当前浅克隆无完整 git 历史可证。
- **当前证据**：README img src；PyPI JSON（name semarun→semanticrun，0.1.0 描述 API 为 SemarunRuntime/ContinuationPolicy 与当前代码不一致）；仓库 301 重定向链（ysharmcode/SemaRun → vogue84/SemanticRun）
- **缺失证据**：完整 commit 历史（浅克隆 depth=1）；改名时间线
- **验证路径**：全量克隆看 reflog/commit 序列；GitHub API 查 rename events
