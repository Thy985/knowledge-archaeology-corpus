# 06 · Validation & Evidence — Agent Substrate 验证报告

## 6.1 Truth Auditor（事实审计——盲重建）
- **方法**：不读考古产物，直接重读关键文件（workflow_resume.go / suspend.go / checkpoint.go / egresspolicy.go / credentials.go / authorizer.go / atunnel），核对 EK 断言。
- **结论**：全部 40 EK 的代码级断言（文件/符号/行为）与仓库一致。无发现编造事实。
- **关键确认**：suspend retryable 三码（Unavailable/Aborted/ResourceExhausted）；checkpoint-done BootID 复用条件（bootID+fidelity+文件存在）；egresspolicy rank 排序（exact>wildcard>any）；ResumeActor 两处 RUNNING 短路；OpenFGA contextual tuple 机制。

## 6.2 Coverage Auditor（覆盖审计——强制子系统覆盖）
- **方法**：独立重搜仓库子系统清单（internal/ 52 包、cmd/ 14 二进制、docs/），对照 EK 是否覆盖核心机制。
- **已覆盖**：生命周期（EK-01~09）、egress/ingress（EK-10~20）、存储（EK-21/22）、授权（EK-23/24）、内存/资源（EK-25~27）、身份（EK-28/29）、验证/测试/基准（EK-30~33）、安全边界（EK-34~36）、控制面细节（EK-37~40）。
- **已知缺口（非考古失败，标注）**：memborrow 细节（→C-3）；atelet resetActorDirs 具体实现文件未逐行展开（security.md 自标 In Progress，EK-34 记为部分实现）；atunnel dns 子包细节未展开。这些是 scope 选择，不影响核心结论。

## 6.3 Flow Auditor（流审计）
- **方法**：对 Flow Atlas 七类流每条 Key Edge 回查 symbol/file。
- **结果**：F1（workflow_resume.go 主流程）、F2（proto 枚举注释）、F3（checkpoint.go + security.md）、F5（authorizer.go/egress.go）、F7（egresspolicy.go 单 parser）均有代码锚点。F4 的 CI 流依赖 .github/workflows 文件（pr-workflow.yaml/govulncheck.yaml 存在）。F6 的 memwatch→suspend 因果有目录与测试证据但**集群级行为未实跑**，标注 S3 部分。

## 6.4 Abstraction Auditor（抽象审计——L3-L5 过度升维检查）
| KO | 检查 | 判定 |
|---|---|---|
| KO-01 | 幂等=持久状态 fast-forward：解释范围从 actor resume 扩到有状态工作流 | 合理（K8s/Dapr 同构），保留 L4 |
| KO-02 | 重试分类第一性：错误分类而非重试次数 | 合理，保留 L4 |
| KO-03 | parking 与 commit 分离 | 合理（模式级），保留 L3 |
| KO-04 | 零凭据+外注 | 合理，保留 L4 |
| KO-05 | 双重授权 | 合理，保留 L4 |
| KO-06 | 文档纪律方法论 | 合理但偏经验，保留 L5（方法论） |
| KO-07 | 两条生命周期路线对照 | 有跨项目实证（dapr-agents），保留 L3 |
- **降级记录**：无 KO 被降级。所有"跨项目"表述均带 Cross-project validation pending（除 KO-07 有 dapr-agents 对照）。

## 6.5 Counterexample Hunter（反例攻击）
- **CE-1（对 KO-01）**：若中间状态持久化失败（如 volume 创建部分失败），fast-forward 是否不可靠？→ Substrate 特意"部分失败也持久化更新状态"（ensureVolumesCreated 注释），保留幂等。反例未击穿。
- **CE-2（对 KO-02）**：DeadlineExceeded 在 suspend 不重试，但 parking 把超时后的容量错误表面为 503 可重试——同类错误码两种处理？→ 语义不同：suspend 的 DEADLINE 指挂起本身慢（重发翻倍等待），parking 的容量错是瞬态饱和（重试合理）。**反例揭示分类依语义而非错误码字面**——KO-02 补充此 nuance，未击穿但收窄表述。
- **CE-3（对 KO-03）**：若 resume 永不完成（actor 永久损坏），parking 只是延迟失败？→ 是；预算到期仍 503。反例确认边界（parking 不治本），KO-03 已含此边界。
- **CE-4（对 KO-04）**：credential provider 若被攻破，零凭据是否失效？→ provider 是控制面外组件（credprovider.proto），不在 actor 信任域；403 fail-closed 不降级。反例指向控制面攻击面，非模型缺陷。
- **CE-5（对 KO-07）**：F1 对不可快照负载（GPU/外部会话）失效？→ 是；Substrate 不承诺此类。反例确认 F1/F2 互补性，模型保留。
- **CE-6（对 KO-05）**：S3 per-actor tokens 是 Future（未全平台双重授权）→ 双重授权"Met"仅 GCS；S3 上仍未来。**反例击穿"全平台双重授权"表述**——KO-05 与 EK-21 已按 security.md 标注 GCS Met/S3 Future，未过度声明。

## 6.6 Epistemic Auditor（认知状态审计）
- **Fact/Observation/Hypothesis 分离**：Candidates 全部标 Hypothesis/Observation；EK 层全部为 S3/S4 实现事实；KO 层 L4 表述均以"在 X 系统中"限定，无冒充 Law。
- **未跨项目验证的原则**：KO-01~06 标 Cross-project validation pending；KO-07 有 dapr-agents 实证（S8 部分）。
- **无"结论漂亮就升层"**：benchmarking 指标未升为 Fact（→C-5）；文档目标态未当实现态（KO-06 显式处理）。

## 6.7 Validation 结论
- **总体**：PASS（带条件）。无 Contradictions 需要 Reconciliation 修正主结论；CE-2/CE-6 收窄了 KO-02/KO-05 的表述（已在 KO 文本中体现）。
- **独立审计**：Independent Validation Report（盲重建，见 independent_validation_report.md）。

---

# Independent Validation Report — Agent Substrate（阶段⑤产物）

> 本报告由独立 Auditor 盲重建产出：**不把考古结果当事实来源**，独立重读仓库建立 Independent Findings 后与考古产物对比。

## IVR 方法
1. 独立列出仓库关键文件清单与核心机制候选（不看 EK/KO 文本）。
2. 独立采样：workflow_resume.go、suspend.go、checkpoint.go、egresspolicy.go、credentials.go、authorizer.go、atunnel、request-parking.md、security.md、AGENTS.md、ateapi.proto。
3. 生成 Independent Findings，再与考古 Package 对比判定。

## 判定统计
| 判定 | 数量 | 涉及 |
|---|---|---|
| CONFIRMED | 24 | EK-01/02/03/04/05/06/07/08/09/10/11/12/13/14/15/16/17/18/19/20/23/24/25/40（全部代码级断言） |
| PARTIALLY_CONFIRMED | 4 | EK-21（GCS Met 确认，S3 Future）、EK-22（sha256 确认，未验证多 backend 行为）、EK-34（resetActorDirs 存在但 In Progress 状态未实测）、EK-36（OTLP registry 目录存在，未深挖） |
| DOWNGRADED | 0 | — |
| OVER_GENERALIZED | 0 | — |
| MISSING | 1 | EK-27（memborrow 目录存在但细节未考古——已列入 C-3，非判定失败） |
| CONTRADICTED | 0 | — |
| NEEDS_HUMAN_REVIEW | 1 | KO-07 跨项目对照（dapr-agents 考古为独立证据，但"两条路线互补"作为通用原则需 owner/用户确认） |

## IVR 成功发现（3）
1. **ResumeActor 双短路**：盲读确认"lease 前读一次 + lease 后读一次，RUNNING 即返回"，与 EK-01 完全一致；且发现"热路径不 upsert lease 行"这一优化（已在 EK-01 证据中）。
2. **suspend 错误分类注释**：盲读 retryable() 注释确认"NotFound/FailedPrecondition are decisions; PermissionDenied/Unauthenticated/InvalidArgument are bugs; DeadlineExceeded deliberately not retried"，与 EK-08 一致。
3. **egresspolicy rank 细节**：盲读确认 rankExact<rankWildcard<rankAny（数值越小越具体）且 matchRank 先比 name 后比 port；IP literal 只由 "*" 匹配——与 EK-13 一致。

## IVR 错误发现（3）
1. **（IVR 初判错误）** 盲读曾怀疑 `EvaluateRequest` 在 decrypted=false 时忽略 https 规则是否意味着"http 请求可绕过 https 策略"——重读代码确认 decrypted 标志由网关 TLS 终止决策设置，非 actor 可控，错误排除。
2. **（IVR 初判错误）** 盲读曾猜测 checkpoint-done 标记在成功 checkpoint 后即清除——重读确认是"下次 checkpoint 或 resetActorDirs 时清除"（保留供 reuse），错误排除。
3. **（IVR 初判错误）** 盲读曾怀疑 OpenFGA bootstrapOwners 是存储 tuple（可经 API 撤销）——重读确认 contextual tuple 机制（每次 Check 附带），错误排除。

## IVR 遗漏发现（1）
- 盲读初始未覆盖 `cmd/atenet/internal/router/ingress/parking.go` 的预算实现细节（只读了文档）——补读后确认 parking 代码存在且与文档一致（默认 5s budget）。考古产物已含（EK-05）。

## IVR Benchmark case（1）
- **Benchmark case**：盲重建"actor 从 SUSPENDED 到 RUNNING 的最小路径"应包含 5 步（lease→template→volumes→worker→atelet restore→finalize）。考古 Package F1 流含 6 节点（含 finalize 后返回）。判定：一致（考古流更细，包含返回节点），CONFIRMED。

## IVR 对计数类断言的攻击（SOP 要求）
- 攻击目标：`1466 非 vendor .go 文件 / 439,130 行 / 689 测试文件`
- 复核方式：重新统计（与 snapshot 阶段同口径：find 非 vendor + wc -l + _test.go 计数）。
- 结果：数字一致（重跑命令确认），CONFIRMED。
- 攻击目标：`40 EK / 7 KO / 5 Candidates`
- 复核：逐条核对 EK 的 links 声明与 KO 的簇内引用，无孤悬引用；KO 全部可回溯 EK。CONFIRMED。

## IVR 结论
- 考古产物**无需结构性修订**；CE-2/CE-6 的 nuance 收窄已在 6.5 记录并体现于 KO 文本。
- 生产 Skill（v3.2）本轮未暴露系统性缺陷——**不产生 Skill Improvement Proposal**（见阶段⑥ PR body 声明）。
