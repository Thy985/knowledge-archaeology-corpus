# 06 Validation & Evidence — Archon（ARCH-2026-10-21-001）

## Truth Audit（真值）
- 全部 20 条 EK 直接回溯仓库实盘（文件:行锚）；无"文档当实现"（auth/config.ts 为实际 gate 代码；CHANGELOG 仅用于迭代证据，实现另查）
- CVE-2026-111322 影响范围（0.2.13–0.11.1）为 KnowlegeMap 雷达外部事实（NVD/VulnCheck），非本仓库内容——标注为外部信源，不做仓库事实

## Coverage Audit（覆盖）
- 已覆盖：控制面认证（B01-B05）/所有权（B06-B07）/凭据端点（B08-B09）/隔离（B10-B12）/凭据治理（B13-B16）/gates（B17-B19）/治理自约束（B20）
- 未深读（声明缺口）：workflows 持久化层全貌（child-isolation.ts 内部）、providers 各适配器、web UI、.archon 方向文档全量、chat-slack 插件内部、git worktree provider 实现细节

## Flow Audit（流真实性）
- 七类流全部从代码导出；关键 Edge 已标注可回溯：
  - assertInternalEndpointBind fatal 先于 Bun.serve（index.ts:509-538）
  - requireConversationVisibility 严格路径无降级（api-request-auth.ts:145-152）
  - 身份先于行加载防枚举（authorizeConversation）
  - overlay 同路径合并无翻译层（container.ts:11-16）

## Abstraction Audit（升维诚实性）
- L4 仅 4 个（KO-B01/B02/B04/B05）+ L3 1 个（KO-B03）——无 L5（方法论级不硬凑）
- 所有 L4 均满足"解释范围扩大"论证 + 五条件检查；KO-B03（隔离分层）明确标注与 Box/MXC 的**对照取舍**而非单向结论
- 无单案例→Pattern 跳跃（C-B04/C-B07 保持 Cross-project pending）

## Counterexample Hunt（反例）
- KO-B01 反例：solo 安装（无 GitHub App）合法绑 0.0.0.0——不变量范围限定"有活凭据分发"的端点（已入 KO 正文）
- KO-B03 反例：in-place 默认 = "用户不知情即以为有隔离"的失败模式——文档明示缓解（AGENTS.md + in-place.ts 注释）
- EK-B03 反例：ARCHON_WEB_AUTH_REQUIRED=false（反代已兜底时登录 UI-only 姿态）——合法放宽，但依赖部署层正确

## Epistemic Audit（认知状态）
- Fact/Observation/Hypothesis 严格分离：C-B01~B07 全标 Hypothesis/Observation；无"结论漂亮就升层"
- C-B04/C-B07 为 Cross-project Candidate，标注验证 pending，未写入 Principle
- 文档-实现对照：CHANGELOG 声明（如 #4154 forge 白名单）标注"迭代证据"，实现另查（credential delivery 未深读全部 host 逻辑——列入缺口）

## 质量指标（初版）
- EK 20 条（B01~B20）/ KO 5 条（R1×1 R2×1 R3×2 R4×1）/ Candidates 7 / Flow 7 类 / evidence 引用 ~60 处
- EK links 覆盖率 100%；平均出边 ~1.9；游离 0%；聚合覆盖率 100%
- 反例预算：L3+ KO 各 ≥1 定向反例（2 个内置 + 部署层 1 个）；0 反例 KO 无

## Contradictions 保留
- 无实现内部矛盾确认；已知**文档-实现缺口**：CHANGELOG #4154（forge 白名单）在 delivery.ts 未直接 grep 到对应实现函数（credential host 检查可能在 forge 包）——**标注为待 Auditor 核实点**，不默认成立也不默认排除
