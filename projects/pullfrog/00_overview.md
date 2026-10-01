# PullFrog 考古总览（Overview）

> run_id: `ARCH-2026-10-02-001` · repository: `pullfrog/pullfrog` · commit: `d287f81e93e245845aa25b0916417a4216f37dad`（2026-10-01 18:10:14 UTC）· skill_version: 3.2
> 定位纠偏：候选卡描述（"纯 GitHub Actions 静态审查 bot"）已过时——PullFrog 实为 **agent 编排 meta-harness**：统一驱动 Claude Code / Codex / OpenCode 三种 agent 后端的 GitHub Actions 运行器，核心命题是"在不可信代码上安全地执行可信 agent"。

## 一句话定位

**PullFrog = 权限分层的 agent 编排 harness**：接收 GitHub 事件（PR/Issue/评论/check_suite），把一个托管 agent（Claude Code / Codex / OpenCode 之一）挂到受控的 MCP 工具面 + 沙箱 shell + 生命周期门控上，替用户完成代码评审、实现、修复、冲突解决等编码任务，并把每个 run 的权限边界、凭证生命周期、失败路径、token 用量全部记录回 PullFrog 服务端。

## 为什么值得考古（Knowledge Value）

1. **"可信 agent + 不可信 repo"的安全模型是 Agent-AI 工程的第一难题**。PullFrog 给出了一个完整可运行的答案：双重 token 分层（gitToken 假定可渗漏 vs mcpToken 不可渗漏）、shell 三态权限（disabled/restricted/enabled）、mount/PID 命名空间沙箱、.git 只读 bind、subagent 状态变更门控（PreToolUse/tool.execute.before）、askpass 令牌生命周期。
2. **失败驱动设计的密度极高**：代码注释里锚定了 ~30 个 issue（#844/#862/#891/#906/#938/#964/#1085/#1093/#1120/#1121/#1140/#1146/#1171/#1179 等），每条关键决策都能回溯到一个真实事故。这是"工程知识底座"的富矿。
3. **三层知识配比天然成型**：Project Layer（harness 架构）→ Engineering Knowledge（沙箱/门控/凭证/超时/重试机制）→ Generalized KO（fail-closed、分层最小权限、双超时、盲重建式 subagent 独立等可迁移认知）。
4. **对抗性测试体系**（test/adhoc、agnostic、crossagent 三层）把安全属性变成可执行断言，是"测试揭示行为"的示范。

## 架构总览（三层视图）

```
        GitHub 事件（PR/Issue/评论/check_suite，14 种 trigger）
                          │
        action.yml（node24，post always()）→ entry.ts → main.ts
                          │
   ┌──────────────────────┼──────────────────────────────┐
   ▼                      ▼                              ▼
 权限/凭证层           生命周期编排层                  Agent 层（三选一）
 resolveTokens        normalizeEnv → prep → setup    claude.ts（1425 行）
 （git/mcp/gh/read    → MCP server → agent.run      codex.ts（998 行）
   四类 token）          → post-run gates →          opencode.ts（1616 行，
 wipeRunnerLeak         finalize → usage PATCH       in-process serve harness）
 shell probe（沙箱）
                          │
   ┌──────────────────────┴──────────────────────┐
   ▼                                             ▼
 MCP 工具面（30+ 工具）                       沙箱执行面
 checkout_pr / git / shell / gh /            shell sandbox（unshare /
 push_branch / review / report_progress      sudo-unshare / userns +
  + subagent 门控 + diff-coverage             FS_MOUNTS tmpfs + .git ro-bind
   pre-flight                                 ASKPASS git auth
```

## 三层产物配比

| 层 | 目标 | 实际 |
|----|------|------|
| Project Layer | 1 地图 | 1（本 overview + 01_project-layer） |
| Engineering Knowledge | 40~60 | 42（EK-01 ~ EK-42，全部带 links） |
| Generalized KO | 7~12 | 8（KO-01 ~ KO-08，全部带 aggregation_rule） |
| Flow Atlas | 七类流 | 7（Control/State/Data/Evidence/Authority/Memory/Policy） |
| Candidates | 未验证假设 | 8 |
| Validation | 六 auditor | Truth/Coverage/Flow/Abstraction/Counterexample/Epistemic + Blind Reconstruction |

## 核心发现摘要（5 条）

1. **分层 token 模型是安全性的主干**：gitToken 按"假定可渗漏"授予（push=enabled→contents:write / disabled→contents:read）；mcpToken 按"不可渗漏"授予（contents/pull_requests/issues/checks write + actions read）；ghToken 单独签发且只镜像触发者自己的 repo 角色（低于阈值根本不注册 gh 工具）；xrepo 时另发 readToken。三个 token 生命周期独立、全部 run 结束吊销（utils/token.ts）。
2. **shell 沙箱是纵深防御而非单点**：检测 unshare/sudo-unshare/userns-unshare 三级降级（CI 下全无 → 硬失败），FS_MOUNTS 在命名空间内 tmpfs 覆盖 `/var/lib/pullfrog`（codex auth.json）、`$RUNNER_TEMP/_runner_file_commands`（防 GITHUB_ENV 注入），并对整个 `.git` 做只读 bind——防 agent 种 git filter/hook 在后续 workflow step 触发代码执行（mcp/shell.ts）。
3. **subagent 与 orchestrator 共享工作树 = 状态变更工具必须门控**：2026-05-18 zed-industries/cloud 事故（reviewfrog 调用 checkout_pr 导致 orchestrator push 覆盖无关分支）催生了双层门控——从工具注册的 `mutates` 标志派生子代理禁调用集（单源防漂移），Claude 侧 PreToolUse hook（exit 2 阻断）+ OpenCode 侧 tool.execute.before，同时 checkout_pr/push_branch 内置 runtime backstop（initialHead 不变量 + pushDest 白名单）。
4. **双超时 + 安全网是"agent 失联"问题的完整解法**：outer process-output watchdog（900s，噪声模式过滤 mcp-proxy SSE 重连防僵尸）+ inner 事件静默 watchdog + first-event budget（120s，83 次实测 p50 5.6s / p90 8.4s / max 39s 推导）+ inner kill 后 5min safety-net（先 dispose MCP server 再 forceReject，否则 re-prompt 落在死 MCP 上产出"自信无工具"的 turn，#1085）。
5. **post-run 门控是"可见交付"的质量护栏**：四类 PostRunIssues（stopHook/dirtyTree/summaryStale/unsubmittedReview），Review 模式只认 `create_pull_request_review`（report_progress 不替代），重试预算 MAX_POST_RUN_RETRIES=3；软门（summaryStale）一次性 nudge 不烧预算；硬门（unsubmittedReview）耗尽预算后转 terminal hard-fail——"run 交付了无可见产出的东西 = 失败"。

## 关键争议（留给阶段⑤验证与阶段⑥ reconcile）

- stop hook 当前在 collectPostRunIssues 中被注释禁用（#714 审计：8/9 配置脚本是 foot-gun）——"配置的 hook 不执行"与"用户以为 hook 在把关"之间存在信任落差。
- external GH_TOKEN 路径下 refreshGitToken 不可用（外部 token 无法 re-mint），且 GH_TOKEN 同时充当 gitToken 和 mcpToken——scope 是用户自选，安全模型降级为用户信任假设。
- opencode MCP timeout 660s > checkout_pr 自身 600s 是"宁可等答案也不 abort 重试"的显式选择（#860/#864 git lock 损坏事故），但长时间无响应对 inner watchdog 的语义（lastEventAt 不因 keepalive 刷新）是强依赖。

## 已知代价（Honest Limits）

- 服务端（PullFrog app，含 router/Jev 打分器/credential store）不在本仓库，run-context 边界（`run-context/route.ts` 等）只能从 action 侧推断，无法独立验证。
- `models.ts`（97KB）是超大 catalog，本次只读其结构（routing 判别、isFree 语义），未逐条核对。
- wiki/ 与 docs/ 设计文档在服务端仓库，本文档对设计意图的还原依赖代码注释（本仓库注释密度极高，可部分弥补）。
- 未运行任何测试（只读考古约束）；对抗测试正文已读（fsExfil/tokenExfil 等），但 CI 下真实运行结果不可复现。

## 目录索引

| 产物 | 路径 |
|------|------|
| Project Layer | `01_project-layer/project-layer.md` |
| Engineering Knowledge（EK Graph） | `02_engineering-knowledge/ek-graph.md` |
| Knowledge Layer（KO + aggregation） | `03_knowledge-layer/knowledge-layer.md` |
| Flow Atlas（七类流） | `04_flow-atlas/flow-atlas.md` |
| Candidates | `05_candidates/candidates.md` |
| Validation + Evidence | `06_validation/validation.md` |
| Run Metadata | `06_validation/run_metadata.yaml` |
