# 05 Candidates — 未验证假设

> 不能确认的内容留在 Candidate。Hypothesis 不得冒充 Fact；Cross-project Candidate 不得写成已验证 Principle。

## C-01 记忆=文件仓库路线的跨项目收敛（Cross-project Hypothesis）
- **假设**: OKF（Git-native Markdown+YAML frontmatter）、Grok Build（markdown notes）、Anthropic /mnt/memory、Letta（git-backed MemFS）的"记忆=文件仓库"收敛，可能预示 agent 记忆的下一范式 = 文件系统级记忆。
- **当前证据**: Letta 生产级实现（本考古）；雷达增量 4 条（09-18 扫描）；KnowlegeMap 同构。
- **缺失证据**: 各项目间无直接互操作/继承关系证明；无 benchmark 对比文件路线 vs 向量路线。
- **验证路径**: MemEval/LoCoMo 类评测对文件记忆 vs 向量记忆的对比；跨项目 K3 设计模式提取。

## C-02 符号链接逃逸——realpath 防护已存在，剩余为内核层绕过（Scope-uncertain，Reconciled）
- **假设（修正后）**: 路径墙的 symlink 逃逸风险**已在设计层被 canonicalizeRoot 的 realpath 显式防护**（sandbox-policy.ts 注释：lexical path 经 symlink 会 silently match nothing = 沙箱放行一切）；剩余风险为内核层绕过（bwrap/seatbelt 具体 mount/rule 配置）。
- **当前证据（Reconciliation 新增，IF-02）**: `canonicalizeRoot` 对每个 root realpath 最近存在祖先；策略在传给 seatbelt/bwrap 前全部 canonical 化。
- **缺失证据**: 无针对 symlink 逃逸的自动化回归测试；bwrap/seatbelt 参数细节仍需审阅（C-02 原文盲重建已读，防护确认）。

## C-03 v2 read_only 移除 = 安全演进 or 降级？（Tentative）
- **假设**: v2 用 frontmatter 白名单（name/description）取代 legacy read_only 保护，可能是设计演进（白名单更严）而非降级。
- **当前证据**: v2 只允许两个字段，任何其他字段（含 read_only）都会被拒——语义上"无法声明只读"。
- **缺失证据**: 无 v2 下"如何实现只读文件"的替代机制证据；server 侧 read_only 如何注入 v2 未确认（memory-frontmatter.ts 注释"read_only may exist (from server)"是 legacy hook 语境）。
- **验证路径**: 查 server 端 memory 对象如何映射到 v2 文件。

## C-04 reflection merge 的 reason 可信度（Tentative）
- **假设**: reflection 子代理 auto-merge 产出的记忆变更，其 reason 由子代理生成，可能与主 agent 意图不一致。
- **当前证据**: reflection-merge auto/explicit 模式存在；memory 工具 reason 必填（对子代理同样生效）。
- **缺失证据**: 无 explicit merge 时主 agent 审查流程的具体实现证据；无 reflection 测试断言 reason 内容。

## C-05 post-turn push 失败路径——状态机存在，补偿策略未验证（Reconciled）
- **假设（修正后）**: post-turn push 失败时**有显式状态记录**（MemoryPostTurnSyncStatus = clean/pushed/dirty/conflict/push_failed/skipped，memory-git.ts pushMemory），但 **conflict 恢复/补偿策略未验证**。
- **当前证据（Reconciliation 新增，IF-06）**: pushMemory 直接 `git push -u origin main`；NON_FAST_FORWARD_PUSH_ERROR_RE 提示 fetch first。
- **缺失证据**: conflict 状态下自动 fetch/merge/重试的流程无测试证据；无推送补偿通知机制。

## C-06 记忆约束的"共享记忆"布局（Shared-memory layout）
- **假设**: shared-memory 布局（validateMemoryTreeConstraints 的第三种 layout）可能用于多 agent 协作记忆，且约束校验规则不同。
- **当前证据**: MemoryTreeConstraintsOptions layout = "root-marker" | "legacy-only" | "shared-memory"；installSharedMemoryPreCommitHook 存在。
- **缺失证据**: 未读共享记忆仓库的完整创建/挂接流程；未确认 shared-memory 布局的约束差异。

## C-07 权限模式默认 unrestricted 与治理叙事张力（Needs Review → 已升 D1 争议）
- **假设**: 默认最宽松权限模式（unrestricted）与"harness 严苛治理"印象存在张力；可能是"默认信任、按需收紧"设计，也可能反映个人工具定位。
- **当前证据**: DEFAULT_PERMISSION_MODE="unrestricted"（permissions/mode.ts）。
- **缺失证据**: 无使用数据/telemetry 证明默认模式分布；无文档说明设计动机。

## C-08 mods 全 trusted + 主进程动态加载（NEEDS_HUMAN_REVIEW，Reconciled）
- **假设（升级证据）**: mods 三源（legacy_global/global/agent）**全部标记 trusted:true**（mod-sources.ts），由 mod-engine 经 `await import(...?mod=mtimeMs)` **主进程动态加载**（createRequire from runtime），无可见 mod 沙箱——与记忆子代理 fail-closed 沙箱形成鲜明对比。
- **当前证据（Reconciliation 新增，IF-08）**: mod-sources.ts trusted:true ×3；mod-engine.ts:1452 动态 import；DEFAULT_MOD_CAPABILITIES 全开。
- **缺失证据**: 无 mod 沙箱/权限最小化流程；无第三方 mod 信任边界文档。
- **处置**: **需 Owner 人工确认**——这是有意设计（可信插件模型）还是待加固面。
