# Candidates — 未验证假说 / 跨项目候选（hermes-agent @ fba4cb1）

> 不能确认的内容留在 Candidate，不升 KO。每条标注：L3 模式假设 / 当前证据 / 缺失证据 / 验证路径。

## C-01 记忆冻结 vs 即时生效的边界
- **Hypothesis**：记忆"冻结快照 + 下 session 生效"适合长会话 cache 敏感场景；但"即时生效"（letta post-turn push）适合高频指令变更场景——两者是同一谱系两端而非优劣。
- 当前证据：hermes F-03/F-22（冻结 + cache 神圣）；letta run EK-05（post-turn push）
- 缺失证据：同一任务负载下的端到端成本/正确性对比（cache 节省 vs 记忆滞后代价）
- 验证路径：同 benchmark 双实现跑记忆依赖任务，比较 token 成本与 recall 正确率
- 状态：Cross-project Hypothesis（两项目证据，未做联合实测）

## C-02 技能生命周期自动化的可迁移性
- **Hypothesis**：Curator 式"空闲触发 + 只归档不删除 + pinned 绕过"是 agent 技能维护的普适安全模式。
- 当前证据：hermes F-11/F-12（Curator 不变量）；letta run（memory-subagent 反射写记忆——维护动作的另一种实现）
- 缺失证据：长期运行（>3 月）后的技能库质量测量；误归档率
- 验证路径：长周期运行 + 归档审计；对照无 Curator 基线
- 状态：Tentative Pattern（单项目强实现 + 跨项目概念对照）

## C-03 供应链信任分级谱系
- **Hypothesis**：技能供应链防线存在"格式门（letta frontmatter）→ 信任门（hermes 分级扫描）→ 证明门（SkillFortify 形式化 / TEE attestation）"三层光谱，成熟度递增且互补。
- 当前证据：hermes F-07/F-08（信任门）；letta run EK-15（格式门）；corpus skillfortify（证明门）；agent-firewall 卡（Proof-of-Guardrail TEE）
- 缺失证据：三层防线在同一威胁下的横向评测（注入样本集）
- 验证路径：构造统一注入/恶意技能样本集，三实现各自检测率对比
- 状态：Cross-project Hypothesis（多项目证据，未评测）

## C-04 SQLite WAL 兼容问题普适性
- **Hypothesis**：NFS/SMB/FUSE/WSL1/ZFS 的 SQLite WAL 故障（locking protocol/-shm 损坏）是跨项目普适工程问题，任何本地优先应用都应内建 WAL 回退。
- 当前证据：hermes F-14（WAL 兼容回退，带具体标记串）
- 缺失证据：其他项目（如 dsh-memory-evolve SQLite 面）是否遭遇同类问题
- 验证路径：检查 corpus 其他 SQLite 项目；跨项目 issue 检索
- 状态：Tentative Pattern（单项目强证据 + 普适工程常识支撑）

## C-05 provider 并发隔离的 fail-closed 语义
- **Hypothesis**：contextvars 绑定 + "空 context 落默认 profile 或 secrets fail-closed"是插件后台任务的正确并发契约。
- 当前证据：hermes F-27（ctx_bound/spawn_context_thread）
- 缺失证据：跨语言（TS 插件面）是否同构；真实并发竞争下的泄漏案例
- 验证路径：构造 context 丢失场景测试 secrets 泄漏面
- 状态：Hypothesis（单项目 S3）

## C-06 checkpoint shadow git 的存储成本
- **Hypothesis**：共享 shadow git store（跨项目 blob 去重）在长期运行下磁盘成本显著低于 per-project repo。
- 当前证据：hermes F-17（pre-v2 one-repo-per-workdir 曾 ~40MB/项目 → v2 共享 store）
- 缺失证据：生产级长期数据（大项目 + 高频快照）
- 验证路径：对比新旧方案的磁盘占用实验（代码注释已给 pre-v2 数字）
- 状态：Hypothesis（设计内证据，未独立实测）

## C-07 人格文件 user_authored 例外的安全权衡
- **Hypothesis**：user_authored 注入只警告不 block 会引入"用户自己误写注入文本"的残余风险面。
- 当前证据：hermes F-19（#112570 权衡：身份文件完整性 > 严格阻断）
- 缺失证据：该例外被利用的真实案例
- 验证路径：威胁建模演练；检查 issue 记录
- 状态：Hypothesis（设计决策已知，风险未量化）

## C-08 记忆 provider 单外部约束的长期可扩展性
- **Hypothesis**：单外部 provider 约束在记忆聚合需求增长时成为瓶颈（多记忆源合并需要多 provider）。
- 当前证据：hermes F-26（明确拒绝多 provider：schema bloat/conflicting backends）
- 缺失证据：真实多源记忆场景的需求强度
- 验证路径：跟踪 hermes memory provider 生态发展
- 状态：Hypothesis（设计取舍，未来演进不确定）

## C-09 HERMES_STATE_DB_GUARD_BYPASS 逃逸面（Reconciliation R-6）
- **Hypothesis**：`HERMES_STATE_DB_GUARD_BYPASS` 环境变量可绕过生产状态库 guard——其风险面取决于该变量是否属设计内调试逃生舱。
- 当前证据：hermes_state_guard.py:23（`_STATE_DB_GUARD_BYPASS_ENV = "HERMES_STATE_DB_GUARD_BYPASS"`）
- 缺失证据：该变量的文档化用途、发布构建中是否可被外部注入
- 验证路径：Owner 确认设计意图 + 威胁建模（环境变量注入面）
- 状态：Hypothesis（NEEDS_HUMAN_REVIEW）
