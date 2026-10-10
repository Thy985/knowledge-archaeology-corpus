# 02 Engineering Knowledge（EK Graph）— deepseek-harness v0.2.1-alpha.2

> EK 是推理原材料（无损底座），不因形成 KO 删除。每条含证据来源（`repo/` 相对路径）。演进标注：**STABLE**（基线已记录且维持）/ **CHANGED**（基线有、0.2 演进）/ **NEW**（0.2 新增）。links 六类边：mechanism/subsystem/causal/dependency/constraint/contrast。

## EK 清单（40 条）

### A. 会话与持久化（session & persistence）

- **EK-01（STABLE）session log append-only 事实源**：会话日志是不可变追加事实源；`deriveMessages()` 从它投影模型历史；"model-visible means logged"——每个模型请求必须可从日志重建。证据：`docs/architecture.md` §Session log；`packages/core/session/src/invariant.ts`。links: [subsystem:EK-02, EK-03, EK-09][dependency:EK-31]
- **EK-02（CHANGED）会话格式版本化治理**：`SESSION_FORMAT_VERSION` 从基线 pinned 0 演进为 **4**；`latestFinalizedVersion: 4`、`latestReleasedVersion: 4`（evidenceTag `dsh-v0.2.0-rc.2`）；alpha/beta/rc 发布即触发 released-format 义务；版本为**单调整数无 major/minor**；bump 条件=现有 discriminator 无法阻止不安全解释（structural difference alone 不够）；`ignorable` marker 治理未知事件类型而非任意嵌套 payload；stored==current 时 adjacent migration chain 不运行（Reconciliation I-16 补录，兼容性审查记录于 `.agents/notes/implemented/process/2026-10-08-session-reader-compatibility-review.md`）。证据：`packages/core/session/src/types.ts:89` + 设计注释；`docs/session-format-status.md`。links: [causal:EK-31][constraint:EK-03][mechanism:EK-10]
- **EK-03（NEW）released session format 迁移纪律**：迁移用 adjacent version-named successor（`session.vN.jsonl`），提交世代永不 rename/replace/delete；每个 adjacent migration 包只做 vN→vN+1；前代不隐含 fallback/downgrade 支持。证据：`docs/architecture.md` §Session log；`AGENTS.md`。links: [constraint:EK-02][mechanism:EK-10][subsystem:EK-01]
- **EK-04（NEW）assistant/attempt 保留未提交尝试**：`assistant/attempt` 保留 settled 失败/重试/取消/流错误尝试而不增加模型历史；`assistant/message` 内嵌精确紧凑流。证据：`docs/architecture.md` §Session log。links: [mechanism:EK-07][causal:EK-05]
- **EK-05（NEW）failed steps 记录 missing tool results**：失败的 step 记录缺失的工具结果（不伪装成功）。证据：`docs/architecture.md` §Turn flow；`packages/core/agent-loop/README.md`。links: [mechanism:EK-04][contrast:EK-12]

### B. 循环与控制面（loop & control plane）

- **EK-06（STABLE）agent-loop turn/step 生命周期**：step=一次模型请求+其工具调用；turn=零或多 step，opening 前 claim 首个输入、closing 于无所欠。证据：`docs/architecture.md` §Turn flow。links: [mechanism:EK-07, EK-08][subsystem:EK-09]
- **EK-07（CHANGED）agent/pre-step 决定 accepted input**：listener 可 rewrite/reject claim 的消息；rejected 或空首 claim 关闭 durable turn 而无 step；enter 决策可设 `startsRequestSeries`。证据：`docs/architecture.md` §Turn flow。links: [mechanism:EK-06][causal:EK-08][dependency:EK-14]
- **EK-08（NEW）prepareCall 实际路由 + 取消不提交**：`agent/request` 与 `prepareCall()` 在 system prompt 与 accepted users 提交前解析实际路由；两个异步阶段任一取消则二者都不提交（cancellation commits neither system nor users）。证据：`docs/architecture.md` §Turn flow。links: [mechanism:EK-07][causal:EK-06][constraint:EK-05]
- **EK-09（STABLE）事件三域**：session 事件（durable 事实）/ agent 事件（活 Agent：inbox/step/status/request/validation/continuation）/ capability 事件（fs/*、tools/*、telemetry/* 挂策略与适配器）。证据：`docs/architecture.md` §Events。links: [subsystem:EK-01, EK-06][mechanism:EK-14]
- **EK-10（NEW）系统提示作为 surface node**：system-prompt-as-surface-node 决策——prompt 只以 `system/message` 历史存在；空渲染清空所有活动 system 节点；capable 路由在缓存历史后追加非空更新（含工具更新）；incapable 路由与新请求系列在首 system 节点合并非空 prompt 文本。证据：`docs/architecture.md` §Turn flow；`.agents/notes/implemented/architecture/2026-09-02-system-prompt-as-surface-node.md`。links: [mechanism:EK-02][dependency:EK-14][constraint:EK-08]
- **EK-11（STABLE）projection seam mandatory**：`dsh-session-projection` 拥有 `ctx.sessionProjections`；注册单元增量折叠已提交事件；host 读取器在激活时要求服务或显式失败；agent loop 注册共享 turnBoundary 状态。证据：`docs/architecture.md` §Session log；`.agents/notes/…/2026-08-19-session-projection-mandatory-seam.md`。links: [subsystem:EK-01][mechanism:EK-02]
- **EK-12（NEW）guard 家族：loop-hygiene**：`repeat-tool-reminder`（per-agent repeat-call detector，enrich post-execute decision，提醒模型换策略或结束）+ `timeout-policy`（cooperative tool-call timeout enforcer，工具声明 timeoutMs，deadline/timeoutOf，嵌套外层 deadline 作用域）；均默认随 base bundle 启用。证据：`packages/guard/README.md`；`packages/guard/repeat-tool-reminder/src/index.ts:2-5,16,24`；`packages/guard/timeout-policy/src/index.ts:2-15,21`。links: [mechanism:EK-05][subsystem:EK-13][contrast:EK-05]
- **EK-13（CHANGED）工具把关管道 + 显式 bypass 路径**：作用域工具注册表 + 把关流水线（PTC mode 传输、策略前后处理、单调守卫）；`tools/pre-execute`/`execute`/`post-execute` 三 waterfall（Scoped<ToolRuntime>）；**"Policy replacements remain authoritative; pipeline failures that bypass post-execute skip projection"**——管道失败绕过 post-execute 时跳过投影（Reconciliation I-15 补录）。证据：`docs/architecture.md` §Core packages；`packages/core/tools/src/index.ts:154,165,177,239-242`。links: [dependency:EK-14][subsystem:EK-12][constraint:EK-17]
- **EK-14（NEW）PTC runtime seam**：`ctx.ptcRuntime.resolve(request)→PtcRunSpec` / `run(spec)→PtcRunResult`；program 作为 async function body（顶层 await/return 可用）；lossless-JSON result/logs/error（error 是 result 的字段不是 reject）；bindings 命名可移植（保留字排除）；**language/isolation 描述符不承诺安全边界**。证据：`packages/ptc-runtime/ptc-runtime/src/index.ts:143,150`；`src/types.ts:70,100,140`；README。links: [mechanism:EK-13][dependency:EK-17][contrast:EK-15]

### C. 权限、沙箱与安全（authority & sandbox）

- **EK-15（CHANGED）sandbox 组重构**：基线 e2b 包组移除；sandbox 组现含 sandbox / sandbox-local / sandbox-policy / sandbox-windows-acl；`ctx.sandbox` 后端 + 消费者包装 argv。证据：`packages/sandbox/`（find 实测无 *e2b*）；`docs/architecture.md` §Capability seams。links: [subsystem:EK-17][contrast:EK-14][mechanism:EK-16]
- **EK-16（NEW）SAFETY.md 显式安全声明**：dev-preview 未审计、不得当生产；沙箱/审批/权限控制**不保证隔离**；"Do not rely on DeepSeek Harness as the sole security control for untrusted workloads"；最小权限/一次性 VM/备份。证据：`SAFETY.md`。links: [constraint:EK-15, EK-17, EK-14][mechanism:EK-18]
- **EK-17（STABLE）approval / permission-presets / credentials seam**：审批提示、权限预设、凭据管理全套 seam（EP-002 权限面）。证据：`docs/architecture.md`；`packages/credentials/`、`packages/approval*`（未全列）。links: [subsystem:EK-15][dependency:EK-13]
- **EK-18（STABLE）应用启动纪律**：`verify-application-entrypoints.ts` 把每个 package bin/可执行源/根 demo/start:web 显式分类，**拒绝绕过 dsh 的 Node 应用路径**；仅命名 dsh profiles 可启动应用；`MANIFEST_BIN_ALLOWLIST` 仅 `apps/cli`（dsh）+ `packages/experimental/webworker-packer`（build-only 例外，Reconciliation I-17 补录）；`EXECUTABLE_SOURCE_ALLOWLIST` 为每个可执行源赋显式角色（launcher / test-only driver / build-only wrapper / private packaging-only runtime dispatcher）。证据：`scripts/verify-application-entrypoints.ts`；`docs/architecture.md` §Application launch。links: [constraint:EK-19][mechanism:EK-16]
- **EK-19（NEW）动态扩展受控**：extensions 组——cordis-host-runner 在 `node:vm` realm 暴露运行时检查与进程级动态定义（重启消失）；**no model tool creates dynamic definitions**；持久安装唯一通道是 Plugin Manager。证据：`packages/extensions/cordis-host-runner/README.md`；`packages/extensions/tool-cordis/src/index.ts:23,42`。links: [constraint:EK-18][mechanism:EK-16][contrast:EK-28]

### D. 持续工作控制面（continuous-work control plane，全 NEW）

- **EK-20（NEW）goal 服务**：每会话至多一个持久目标（create/edit/pause/resume/complete/block/clear）；CAS 更新拒绝 stale（不能改 objective/maxGoalRounds）；默认 round cap 256；blocked 目标保留稳定 policy code；**goal 状态持久但 continuation permission 进程级**（不持久化续跑权限）。证据：`packages/goal/goal/README.md`；`src/fold.ts:102-112,187-188,215,221`。links: [mechanism:EK-21, EK-22][subsystem:EK-23][constraint:EK-06]
- **EK-21（NEW）jobs 后台任务注册表**：`<kind>-N` 稳定 id；owner=启动它的 agent session（另一 agent 不能读/停；**fence 是授权不是 secrecy**——id 可预测）；状态 running/stopping/completed/killed/failed；output ring（stdout/stderr→model、log→observers 只读）；settlement→in-session notice 免轮询；byte cap 可选。证据：`packages/jobs/jobs/README.md`；`src/archive-admission.ts:25-35,45`。links: [mechanism:EK-20, EK-22][constraint:EK-06]
- **EK-22（NEW）schedule 持久提醒**：六 selector（after_seconds/at/every_seconds/daily/weekly/cron）；cron 五字段 Vixie 方言 + **canonicalization**（存储规范化表达式，decoder 拒绝非 canonical）；delivery 需 Host Web Session controller + Session persistence（`session/flush` 确认后才 commit）；archiving 含 active reminders 被拒。证据：`packages/schedule/schedule/README.md`。links: [mechanism:EK-20, EK-21][dependency:EK-11][constraint:EK-39]
- **EK-23（NEW）deliverables 交付记录**：`present` 工具声明最终交付文件（注册 ctx.tools）；workspace-changes 从 git 工作树快照记录每 turn 变更文件与行数（ctx.workspaceChanges，听 session/event 追加 workspace/changes）；**仅客户端读**的 durable session 事件。证据：`packages/deliverables/README.md`。links: [subsystem:EK-24][mechanism:EK-01][constraint:EK-25]
- **EK-24（NEW）spill 存储**：`ctx.spillStore.saveText()` 返回 opaque locator + 精确字节数 + 检索指引；`dsh-spill-policy` 把超大文本/图像工具结果变 bounded preview + locator（maxInlineTokens 12500 示例）；无 retention/replacement/retrieval/search 操作。证据：`packages/spill/spill/README.md`。links: [mechanism:EK-23][dependency:EK-01]
- **EK-25（NEW）computer-use / browser-use 单 provider 注册**：一次一个 desktop driver / browser backend；加载另一个 provider 失败并带名字；provider 自带工具与操作；服务本身无 model-visible tools。证据：`packages/computer-use/computer-use/README.md`；`packages/browser-use/browser-use/README.md`。links: [mechanism:EK-28][subsystem:EK-23][constraint:EK-18]
- **EK-26（NEW）workspace registry**：`ctx.workspaceRegistry` 持久项目目录列表 + 会话归属；隐藏会话不删历史；移除目录建新项目；**模型不可见**（无 prompt/request-context 成本）；需 session persistence + storage 后端。证据：`packages/workspace/workspace/README.md`。links: [subsystem:EK-23][dependency:EK-11]

### E. 组装、可观测与互操作（composition, observability, interop）

- **EK-27（STABLE）profile/bundle 分层组装**：运行 dsh=boot 时按序层组成的插件树；profile（web/headless/sdk/sdk-minimal/acp）列出 bundle 栈 + out-of-tree 插件 + 用户 cordis.patch.yml；patch 按行 id 整体替换 config 或插入新行；preset patch 在声明内 config.plugins 应用。证据：`docs/architecture.md` §Profiles and bundles。links: [subsystem:EK-28][mechanism:EK-19]
- **EK-28（STABLE）能力 seam 三角色**：Service Definition / Service Provider / Consumer；一个 provider 替换改变整个产品（fs/subprocess 共享执行世界 → 指向远程沙箱则 Bash/PTY/LSP 一起搬走）；subagent providers 同样可变。证据：`docs/architecture.md` §Capability seams。links: [mechanism:EK-27][contrast:EK-25][subsystem:EK-29]
- **EK-29（CHANGED）Experimental Agent Teams**：opt-in `ctx.agentTeams` 协调，durable roster + task state + 经 continuable subagents 的 direct inbox 消息。证据：`docs/architecture.md` §Capability seams。links: [subsystem:EK-28][mechanism:EK-30]
- **EK-30（STABLE）subagent providers**：spawn/fork/ACP/Codex/Claude Code 驱动的 subagent（基线已记录）。证据：`docs/architecture.md`；`packages/subagent/`。links: [mechanism:EK-29][subsystem:EK-28]
- **EK-31（CHANGED）JSONL 世代文件布局**：v0 用 `session.jsonl[.zstd]`，v1+ 用 lowercase `session.vN.jsonl[.zstd]`；JSONL provider 拥有物理 framing/压缩/世代选择/独占发布。证据：`docs/architecture.md` §Session log。links: [mechanism:EK-02, EK-03][subsystem:EK-01]
- **EK-32（NEW）working directories**：提供用户上下文与执行路径而不改变原始项目身份、沙箱写根、既有进程目录。证据：`docs/subsystems/working-directory.md`；`docs/architecture.md`。links: [dependency:EK-15][subsystem:EK-28]
- **EK-33（STABLE）session telemetry OTEL**：session-telemetry-otel（OpenTelemetry 后端）。证据：基线记录 + `packages/telemetry/`。links: [mechanism:EK-34][subsystem:EK-23]
- **EK-34（NEW）Python SDK jsonrpc benchmark**：BENCHMARK.md 指引 Python SDK jsonrpc-agent 最小变体跑 benchmark，独立 workspace/session id；runtime wheel 打包 dsh CLI。证据：`BENCHMARK.md`；`docs/architecture.md` §Application launch。links: [mechanism:EK-27][subsystem:EK-33]
- **EK-35（STABLE）MCP 独立组**：基线内置 mcp，0.2 为独立包组（`packages/mcp/`）。证据：`packages/mcp/`。links: [subsystem:EK-36][mechanism:EK-30]
- **EK-36（STABLE）ACP**：Agent Client Protocol（自动化专用 ACP server profile）。证据：`packages/acp/`；`docs/architecture.md`。links: [mechanism:EK-35][constraint:EK-18]

### F. 失败、配置与边界（failure, config, boundaries）

- **EK-37（CHANGED）HMR config-only**：base 启用 config-only `dsh-hmr`；headless/SDK/ACP 禁用；sdk-minimal 省略；profile patches 覆盖默认。证据：`docs/architecture.md` §Profiles and bundles。links: [dependency:EK-27]
- **EK-38（NEW）guard 动机：卡死/挂死反模式**：repeat-tool-reminder 动机=stuck loop 烧时间/token；timeout-policy 动机=hung call 挂死会话。证据：`packages/guard/README.md`。links: [causal:EK-12][contrast:EK-05]
- **EK-39（NEW）schedule 组装边界**：schedule 不能 headless/sdk-only 单独挂载（delivery 需 Host Web Session controller + Session persistence backend + session/flush 确认）；deliveryHistoryDays 默认 30 / deliveryHistoryRecords 默认 200。证据：`packages/schedule/schedule/README.md`。links: [constraint:EK-22][dependency:EK-11]
- **EK-40（NEW）sdk-minimal 例外**：唯一不应用 dsh-base 的 bundle（拥有完整显式 SDK 树）；Python 最小示例选择 sdk-minimal profile。证据：`docs/architecture.md` §Profiles and bundles。links: [contrast:EK-27][subsystem:EK-34]

## EK Graph 度量

- 总数：40 ｜ links 声明：40/40（100%）｜ 游离 EK：0 ｜ 平均出边：40 条边以上（每条 ≥1）
- 边类型覆盖：mechanism(≥10) / subsystem(≥12) / causal(≥6) / dependency(≥8) / constraint(≥10) / contrast(≥6)

## 演进标注统计

- STABLE：EK-01/06/09/11/13/17/18/27/28/30/33/35/36（13）
- CHANGED：EK-02/07/15/29/31/37（6）
- NEW：EK-03/04/05/08/10/12/14/16/19/20/21/22/23/24/25/26/32/34/38/39/40（21）

> 说明：EK-07 基线已有 pre-step 机制，0.2 文档化 startsRequestSeries/rewrite-reject 细节更完整 → CHANGED；EK-15 因 e2b 移除 → CHANGED；EK-02/31/37 因版本化/布局/HMR 明确化 → CHANGED。
