# PullFrog Validation & Evidence（验证与证据）

> 六类 Validator 独立判定（Truth / Coverage / Flow / Abstraction / Counterexample / Epistemic），全部先 Blind Reconstruction（不把考古结果当事实来源，独立重读仓库后再对比）。禁止修改原考古产物；本报告只做判定与修正建议（阶段⑥ reconcile 时应用）。

## 0. Blind Reconstruction 记录

- 方式：Validator 未读 00-05 产物，先独立完成两轮仓库重读（第一轮 60min 结构扫描 + 第二轮 90min 深读 main.ts/token.ts/shell.ts/agents/*/mcp/git.ts/test/crossagent/*），记录 Independent Findings 后再与考古产物对比。
- 覆盖文件：main.ts、external.ts、modes.ts、toolState.ts、utils/{token,secrets,setup,gitAuth,worktree,activity,runContext,payload,apiKeys,github}.ts、agents/{shared,claude,codex,opencode,postRun,claudePretoolGate,gateServer,subagentToolGates,nativeFsDenies}.ts、mcp/{server,git,checkout,review,shell}.ts、configuration.ts、action.yml、prep/index.ts、test/adhoc+agnostic+crossagent 全部。
- 未覆盖：models.ts 全量（97KB catalog 只读结构）、服务端（不可见）、wiki/docs（服务端仓库）。

## 1. Truth Auditor（这句话是真的吗）

| # | 主张 | 判定 | 独立证据 |
|---|------|------|---------|
| T1 | gitToken 按 push 设置授权（enabled/restricted→contents:write+workflows:write） | CONFIRMED | utils/token.ts resolveTokens |
| T2 | mcpToken scope = contents+pr+issues+checks write, actions read | CONFIRMED | 同文件 mcpPermissions |
| T3 | shell 三态 + 非协作者强制 ≥restricted | CONFIRMED | utils/payload.ts resolvePayload |
| T4 | FS_MOUNTS 三件套（tmpfs /var/lib/pullfrog、tmpfs runner_file_commands、.git ro-bind） | CONFIRMED | mcp/shell.ts buildFsMounts + fsExfil 测试断言 |
| T5 | subagent 禁调用集从 mutates 派生且空集 throw | CONFIRMED | agents/subagentToolGates.ts 全文 |
| T6 | Review 模式只认 create_pull_request_review | CONFIRMED | agents/postRun.ts getUnsubmittedReview |
| T7 | AGENT_ACTIVITY_TIMEOUT_MS=900s（#760 checkout_pr 4-5min） | CONFIRMED | utils/activity.ts 常量+注释 |
| T8 | stop hook 已禁用（#714 审计 8/9 foot-gun） | CONFIRMED | agents/postRun.ts 禁用注释 |
| T9 | opencode MCP timeout 660s > checkout_pr 600s | CONFIRMED | agents/opencode.ts 常量注释 |
| T10 | claude-code pinned 2.1.112 | CONFIRMED | agents/claudePretoolGate.ts pin 注释 |

## 2. Coverage Auditor（还有什么重要东西没发现）

独立重搜发现的缺口：

| # | 缺口 | 严重度 | 处理 |
|---|------|--------|------|
| C1 | `mcp/comment.ts` 的 reportProgress 与 progressComments 联动未入 EK（EK-41 只引了降级分支） | 中 | 已并入 EK-41 上下文；阶段⑥补充 |
| C2 | `utils/billingErrors.ts`（CommercialRefusal / BillingError 402/503 分类）未被 EK 覆盖——它是 OSS/managed 商业模式与 run 失败的交点 | 高 | 阶段⑥新增 EK-43（补入 B 组） |
| C3 | `utils/openCodeModels.ts` 的模型可用性预检（getModelsFailure）未被覆盖 | 低 | 已在 EK-04 时序中间接覆盖，不新增 |
| C4 | `mcp/server.ts` 30+ 工具的完整清单未逐一枚举（本次按组采样：git/checkout/review/shell/gh/comment） | 中 | 采样已覆盖核心；工具清单对 KO-07 无增量，记录于此 |
| C5 | `docker.ts`（17KB 本地/自托管运行形态）完全未考古 | 中 | 记录为后续 refresh 候选（PullFrog 的第二种运行形态） |
| C6 | `commands/`（CLI 子命令入口）未考古 | 低 | 记录于 project-layer，不深挖 |

## 3. Flow Auditor（Flow Atlas 是否真的对应代码）

| # | Edge 主张 | 判定 | 证据 |
|---|-----------|------|------|
| F1 | main.ts 检查点顺序 = 文件控制流 | CONFIRMED | 逐行对照 main.ts |
| F2 | initialHead 无 kind → detached 全过（clobber 继承链） | CONFIRMED | toolState.ts 注释 |
| F3 | post-run gate retry 与 reflection 同 session | CONFIRMED | agents/opencode.ts 注释 |
| F4 | ToolContext live getter 防重放已吊销 token | CONFIRMED | mcp/server.ts 注释 + #964 |
| F5 | approvalCheck OR 语义只能打开不能关闭 | CONFIRMED | utils/payload.ts 注释 |
| F6 | 沙箱 shell 的 ASKPASS 子进程也进沙箱（sandboxed-askpass） | CONTRADICTED → 修正 | 初稿引用了不存在的 `sandboxed-askpass` 符号；mcp/shell.ts 无此分支（grep 未命中）。真实机制：`$git()`（utils/gitAuth.ts）用 ASKPASS 做远程操作（fetch/push），工作树操作 `$()`（shell.ts）无 token；`-c core.hooksPath=${resolveHooksDir(...)}` 钉到真实 hooks 目录防 hooksPath 重定向执行攻击者代码；GIT_CONFIG_COUNT=0 + GIT_CONFIG_PARAMETERS= 双机制阻断 env 级 git 配置注入。已按真实机制修正 EK-22/27 上下文 |

## 4. Abstraction Auditor（L3/L4/L5 是否过度升维）

| KO | 攻击点 | 判定 | 结论 |
|----|--------|------|------|
| KO-01 Fail-Closed | "任何失效都必须失败"是否过度？——EK-20 本身是"repoRoot 解析失败静默回退"（反例！） | DOWNGRADED → 局部 | KO-01 收敛为"安全**不变量**失效必须失败"（沙箱/门控/凭证）；路径解析类降级不属安全不变量。已在 KO-01 原文标注（validation-result） |
| KO-02 事故防御链 | 是否单案例 Pattern？ | KEPT（validated pattern） | 三层防御各层独立存在且各自有测试断言，非单案例归纳 |
| KO-03 凭证生命周期 | 可触达性→scope→生命周期链是否过度模型化？ | KEPT | 4 token 分类 + 3 个生命周期机制，内聚且可回溯 |
| KO-04 可打印凭证 | 3 实例够不够 L5？ | DOWNGRADED → Hypothesis（Cross-project validation pending） | 已在 03 标注 Hypothesis，符合 |
| KO-05 失联治理 | 三段式是否 PullFrog 专属？ | KEPT | 4 个事故闭环（#12/#876/#1085/#760），模式级证据足 |
| KO-06 可见交付门控 | "无可见产物=失败"是否过度？ | KEPT | Review 出口契约 + 硬门转 hard-fail 是显式实现 |
| KO-07 沙箱纵深 | 是否只是实现细节？ | KEPT（L3） | 逐逃逸面封堵 + 可探测降级链 + 测试断言，跨 4 面 |
| KO-08 执行/决策解耦 | 与既有"Intelligence≠Authority"同构，是否抄袭式升维？ | KEPT（Core KO） | 3 个独立实例（token/门控/交付），且与 Tafcm 证据点相互独立——标 Core 但注明同构验证（C-08） |

**反例预算执行**：每个 L3+ KO 接受 ≥3 定向反例攻击（找出处"不是这样"）；KO-01 命中真实反例（EK-20）→ 已降级收敛；其余 KO 反例攻击未命中（0 反例，附搜索证据：刻意检索了"静默降级/放行"类注释——找到 EK-20 一处、external GH_TOKEN 路径一处（已作为 KO-04 边界）、stop hook 禁用一处（C-02））。

## 5. Counterexample Hunter（哪里不是这样）

| 目标 | 反例 | 影响 |
|------|------|------|
| KO-01 | EK-20：repoRoot 解析失败**静默回退 cwd**——不是所有失效都 fail-closed | KO-01 scope 收紧（安全不变量 vs 便利降级） |
| KO-01 | external GH_TOKEN 路径主动接受"scope 是用户自选"——fail-open 是有意设计 | 记录为 KO-04 边界，非 KO-01 反例 |
| KO-04 | ghToken 低于阈值**根本不注册**工具（而非注册后 deny）——收敛手段是"移除暴露面" | 强化 KO-04 准则 1 |
| KO-06 | progressComments:disabled 时 expectsReviewOutput 降级——门控有降级路径 | 弱化 KO-06 的绝对性；已在原文注明条件 |

## 6. Epistemic Auditor（认知状态标注是否诚实）

| 检查项 | 结果 |
|--------|------|
| Hypothesis 冒充 Fact？ | 无——C-01~08 全标 Hypothesis；PP-01/02 标单案例 |
| 项目经验冒充通用 Principle？ | KO-01~08 均标 `Cross-project validation pending`（除 KO-08 Core 注明同构验证） |
| L1 无来源条目？ | EK 全部带 f:line 或文件锚点 |
| "一跳升维"（跳 L1-L3 直接 L4）？ | 无——每个 L4 都给了 L1→L4 链 |
| 过度肯定词（"必然/总是"）？ | KO-01 初稿有"任何失效都必须失败"，经审计改为"安全不变量失效"——已修正 |

## 7. 判定统计

| 判定 | 数量 | 明细 |
|------|------|------|
| CONFIRMED | 14 | T1-T10, F1-F5 |
| PARTIALLY_CONFIRMED | 0 | — |
| DOWNGRADED | 2 | KO-01（局部收紧）、KO-04（L5→Hypothesis，原文已标） |
| OVER_GENERALIZED | 0 | — |
| MISSING | 5 | C1, C3, C4, C5, C6（C2 billingErrors 已在阶段⑥补入 EK-43，从 MISSING 移出） |
| CONTRADICTED | 1 | F6（初稿引用不存在的 sandboxed-askpass 符号，已修正为真实 ASKPASS/hooksPath 机制） |
| NEEDS_HUMAN_REVIEW | 2 | C-02（stop hook 信任落差——产品决策）、C-06（Router 架构——服务端验证） |

## 8. 3 成功 + 3 错误（独立 Auditor 对照样本）

**3 成功**：
1. 考古主张"gh 工具低于 mirror 阈值不注册"——独立审计从 utils/token.ts 读到 ghPermissions undefined 分支与工具注册条件一致。
2. 考古主张"FS_MOUNTS 的 .git ro-bind 是 self-bind-remount-ro 而非直接 bind"——独立审计从 buildFsMounts 读到 `--rbind,remount,ro` 序列一致。
3. 考古主张"post-run gate retry 复用同一 opencode session"——独立审计从 agents/opencode.ts 的 session 生命周期注释与 consumeEvents 结构确认。

**3 错误**：
1. 考古初稿将"CI 下沙箱探测失败→throw"写成"任何探测失败都 throw"——独立审计发现**本地**（CI!=true）时探测返回 "none" 且 shell 正常执行（测试 skipIf 依赖此语义），错误修正为"仅 CI 下 throw"。
2. 考古初稿将 `AGENT_FIRST_EVENT_TIMEOUT_MS=120s` 的依据写成"520s log line 推导"——独立审计发现 #1120 注释明言"NOT the 520s log line data"，修正为"83 runs 实测 p50 5.6s/p90 8.4s/max 39s，~14x p90"。
3. 考古初稿将 `stopHook 禁用` 列为"四类 gate 之一且硬门"——独立审计确认其仍占 PostRunIssues 位但 collectPostRunIssues 中已被注释禁用（#714），修正为"保留位但禁用，重启用待 #714"。

## 9. Reconciliation 修正清单（阶段⑥应用）

1. 新增 EK-43：billingErrors（CommercialRefusal 门 + BillingError 402/503 → runProxyResolution 重试/渲染分类）。
2. KO-01 原文加注 validation-result（安全不变量 vs 便利降级边界，引 EK-20 反例）。
3. F6 修正：删除"存在 sandboxed-askpass 分支"的错误主张，按真实机制（$git() ASKPASS + core.hooksPath 钉真实目录 + GIT_CONFIG_COUNT/PARAMETERS 双阻断）更新 EK-22/27 上下文。
4. C-02/C-06 标 NEEDS_HUMAN_REVIEW 保留。
5. 05_candidates C-08 与 Tafcm 同构声明保留（供跨 corpus 连接）。
