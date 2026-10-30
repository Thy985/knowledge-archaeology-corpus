# Independent Validation Report（IVR）— ARCH-2026-10-31-001

> 独立 Auditor 盲重建报告。方法：不把考古产物当事实来源，独立重读仓库（workflow_resume.go / suspend.go / checkpoint.go / egresspolicy.go / credentials.go / authorizer.go / atunnel / request-parking.md / security.md / AGENTS.md / ateapi.proto / .agents/）建立 Independent Findings，再与考古产物对比判定。**禁止修改原考古产物**（修正走 Reconciliation）。

## 判定统计
| 判定 | 数量 | 涉及 |
|---|---|---|
| CONFIRMED | 24 | EK-01/02/03/04/05/06/07/08/09/10/11/12/13/14/15/16/17/18/19/20/23/24/25/40 |
| PARTIALLY_CONFIRMED | 4 | EK-21（GCS Met 确认、S3 Future）、EK-22（sha256 确认、多 backend 未验证）、EK-34（resetActorDirs 存在、In Progress 未实测）、EK-36（OTLP 目录存在、未深挖） |
| DOWNGRADED | 0 | — |
| OVER_GENERALIZED | 0 | — |
| MISSING | 2 | EK-27 细节（memborrow → C-3）；**`.agents/skills/` AI 治理技能套件未被考古产物覆盖（本报告发现）** |
| CONTRADICTED | 0 | — |
| NEEDS_HUMAN_REVIEW | 1 | KO-07 跨项目对照（dapr-agents 独立证据支持，但"两条路线互补"作为通用原则需 owner 确认） |

## 成功发现（3）
1. **ResumeActor 双短路**：盲读确认 lease 前读一次 + lease 后读一次，RUNNING 即返回；且发现"热路径不 upsert lease 行"优化（workflow_resume.go）。
2. **suspend 错误分类**：盲读 retryable() 注释确认 NotFound/FailedPrecondition=decisions、PermissionDenied/Unauthenticated/InvalidArgument=bugs、DeadlineExceeded 刻意不重试（suspend.go）。
3. **egresspolicy rank 细节**：盲读确认 rankExact<rankWildcard<rankAny、matchRank 先 name 后 port、IP literal 仅 "*" 匹配（egresspolicy.go）。

## 错误发现（3）
1. 盲读曾怀疑 EvaluateRequest 忽略 https 规则可被 http 请求绕过 → 重读确认 decrypted 标志由网关 TLS 终止决策设置、非 actor 可控，错误排除。
2. 盲读曾猜测 checkpoint-done 成功即清除 → 重读确认"下次 checkpoint 或 resetActorDirs 清除"（保留供 reuse），错误排除。
3. 盲读曾怀疑 OpenFGA bootstrapOwners 是存储 tuple 可经 API 撤销 → 重读确认 contextual tuple（每次 Check 附带），错误排除。

## 遗漏发现（1）+ 盲重建新发现（1）
- 盲读初始未覆盖 parking.go 预算实现（只读文档）→ 补读确认与 request-parking.md 一致（默认 5s budget）。
- **新发现（MISSING）**：`repo/.agents/skills/` 存在 6 个 AI 治理技能（agents-md / detect-flaky-tests / post-draft-review / review-crds / security-status-report / triage-issues），考古 Package 未覆盖。这是 AGENTS.md 治理文化的**执行工具层**：
  - `agents-md`：生成/更新 AGENTS.md（token 效率导向）
  - `detect-flaky-tests`：7 天 CI 运行分析 flaky Go tests（含 e2e gVisor/microVM lanes），自动开 issue + draft fix PR
  - `post-draft-review`：PR review 以 GitHub **PENDING draft** 评论提交（human 审后 submit），不直接 publish 给作者
  - `review-crds`：按 K8s API conventions 审 CRD
  - `security-status-report`：读 docs/threats.json 起子 agent 计算每个 threat 质量分，渲染 bar chart
  - `triage-issues`：issue 打标——**propose before applying**（先全量提案用户批准）+ **undo script**（每次运行生成可逆脚本）
  - 治理语义：AI 只做"提案/草稿/分析"，人类做"批准/提交/撤销"；**可逆性 + 人审**是这套技能的统一不变量（与 KO-05 双重授权同构）。
  → 交 Reconciliation：补 EK-41（见阶段⑥修正）。

## Benchmark case
- **最小恢复路径**：盲重建"SUSPENDED→RUNNING 最少 5 步"（lease→template→volumes→worker→atelet restore→finalize，实际 6 节点含返回）→ 与考古 F1 流一致，CONFIRMED。

## 计数类断言攻击（SOP 强制）
| 断言 | 重跑结果 | 判定 |
|---|---|---|
| 1466 非 vendor .go | 1466 | CONFIRMED |
| 439,130 行 | 439130 | CONFIRMED |
| 689 _test.go | 689 | CONFIRMED |
| 3 proto | 3 | CONFIRMED |
| 79 .py（snapshot 口径） | 79 全部 / 76 非 vendor / 3 vendor | **PARTIALLY_CONFIRMED**：口径需注明"含 vendor"→ Reconciliation 修正 snapshot 口径注记 |

## 结论
- 考古产物无需结构性修订；CE-2/CE-6 nuance 已在 6.5 记录并体现于 KO 文本。
- **生产 Skill v3.2 本轮未暴露系统性缺陷 → 不产生 Skill Improvement Proposal**。
- 需 Reconciliation 修正项：①补 EK-41（.agents/skills 治理技能层）；②snapshot 的 .py 口径注记。
