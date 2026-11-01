# Independent Validation Report — ARCH-2026-10-19-001（MXC）

> Auditor 盲重建：不把考古结果当事实来源，独立重读仓库建立 Independent Findings 后对比。禁止修改原考古产物（本报告独立成文，修正经 Reconciliation 阶段落地）。

## 判定统计（10 项独立抽查 + 7 项 Findings）

| 判定 | 数量 | 涉及 |
|---|---|---|
| CONFIRMED | 12 | EK-M01/M02/M04/M06/M08/M09/M12/M13/M14/M15/M16/M19（+ 抽查的 M20） |
| PARTIALLY_CONFIRMED | 3 | EK-M05 / EK-M07 / EK-M17（需补充细节，见下） |
| OVER_GENERALIZED | 2 | EK-M03（时间限定）/ EK-M05（kill_for_timeout 语义） |
| MISSING | 1 | Bubblewrap --die-with-parent 精确动机（EK 家族缺一条） |
| CONTRADICTED | 0 | — |
| NEEDS_HUMAN_REVIEW | 1 | C-M01（组织 Intune 策略合并，未见引擎实现） |

## 独立抽查（Blind Reconstruction 证据）

| # | 抽查事实 | 独立证据（文件:行） | 判定 |
|---|---|---|---|
| 1 | experimental 是调用方函数参数非 JSON 字段 | mod.rs:117-119 `pub fn spawn_one_shot_json(request_json, experimental: bool)`；state_aware.rs:113/266/336/398 均从 `parsed.request().experimental_enabled` 读取 | CONFIRMED |
| 2 | NetworkAction 默认 Deny | policy/network.rs:24-30 `#[default] Deny` | CONFIRMED |
| 3 | config_parser 硬编码 experimental_enabled=false | state_aware.rs:427-430 "config_parser hardcodes `experimental_enabled: false` for every state-aware request"；experimental="in-process equivalent of the executor's `--experimental` flag" | CONFIRMED（强化 EK-M08） |
| 4 | 审计事件仅走诊断 sink | logger.rs:387-390 log_audit_event "written when (and only when) a diagnostic sink is attached"；audit.rs:1-25 双 sink 语义 | CONFIRMED（强化 EK-M07） |
| 5 | 契约版本精确匹配，无语义比较 | mxc_contract/version.rs:4-42 "No semantic version ranges or normalization"; parse_exact 精确拼写 | CONFIRMED |
| 6 | PathCanonical 三态 + fail-closed | filesystem_canonical.rs:17-31 Unknown≠Absent，"callers can **fail closed** when deniedPaths" | CONFIRMED（强化 EK-M15） |
| 7 | network_proxy serde(skip) | policy_identity.rs:50、259-260（注释+代码双重确认） | CONFIRMED |
| 8 | Bubblewrap 默认 unshare 全家 + die-with-parent | bwrap_command.rs:416-441 `--unshare-user/pid/ipc/uts/net` + `--die-with-parent` | CONFIRMED |
| 9 | kill_for_timeout 默认 = kill | sandbox_process.rs:264-282 默认实现 `self.kill()`；wait 注释 "Do not read TimedOut as a tree kill without checking the backend" | PARTIALLY_CONFIRMED（见 E-1） |
| 10 | dispatch 未来扩展（实验性后端路由进引擎） | dispatch.rs:21 "in a later increment, will also route through this engine" | OVER_GENERALIZED 风险（见 E-2） |

## 3 个成功（Auditor 支持性强化）

### S-1 · "配置不能自我授权"是双层防御（CONFIRMED 强化）
- **独立发现**: 不只引擎层 `require_experimental_optin` gate——**JSON 解析器层硬编码 `experimental_enabled: false`**（state_aware.rs:427-430），实验性只能由宿主进程在解析后显式设置。EK-M08 的"experimental 是参数非字段"得到第二层证据
- **影响**: EK-M08 建议补入解析器层防御细节

### S-2 · 审计是"廉价惰性"的（CONFIRMED 强化）
- **独立发现**: log_audit_event 仅在诊断 sink（--log-file / MXC_DIAG_CONSOLE）附着时写（logger.rs:387-390），无 sink 时是廉价空操作——审计基础设施不引入恒定开销
- **影响**: EK-M07 建议补"无 sink 时不产生记录"

### S-3 · --die-with-parent 是"时间线逃逸"防御（CONFIRMED 强化）
- **独立发现**: bwrap_command.rs:432-437 SECURITY 注释揭示精确动机——`--unshare-pid` 单独不够：**bwrap forks，新命名空间 pid1 是子进程而非我们 spawn 的进程**，无该标志时后台化后代可在超时 kill 后存活并继续运行在已拆除的网络强制之外
- **影响**: 建议补一条 EK（Bubblewrap 时间线逃逸防御），与 EK-M06 kill_for_timeout 族互为表里

## 3 个错误/风险点（Auditor 主动攻击）

### E-1 · EK-M05 "kill_for_timeout 语义独立"过度概括（OVER_GENERALIZED）
- **攻击**: 默认实现就是 `self.kill()`（sandbox_process.rs:273-276）；且 wait 注释明示 "Do not read `TimedOut` as a tree kill without checking the backend"——树级终止范围因后端而异（Unix 进程组 / Windows job object / state-aware adapter 只确认前台进程，后代留给沙箱生命周期拥有者）
- **修正**: EK-M05 补充：kill_for_timeout 默认=kill；超时终止的树范围是后端实现契约，不是全局保证

### E-2 · EK-M03 "后端选择唯一所在地"需时间限定（OVER_GENERALIZED）
- **攻击**: dispatch.rs:21 文档预告 "in a later increment, will also route through this engine"——实验性后端的 run-to-completion 将来会路由进引擎
- **修正**: EK-M03 限定"当前版本（1.0.1）"；未来扩展不影响主体（今天唯一实现点成立）

### E-3 · EK 家族缺 Bubblewrap 时间线逃逸条目（MISSING）
- **攻击**: 无 EK 单独记录 `--die-with-parent` 的失败模式动机（超时 kill 后后台后代逃逸）
- **修正**: Reconciliation 新增 EK-M21（Bubblewrap 时间线逃逸防御）

## 遗漏与 Coverage 声明
- **未独立深读**（沿用考古覆盖声明）: LXC/Seatbelt/IsolationSession runner 内部、host/（wxc_host_prep/plm）、.NET/Node SDK 包装层
- **NEEDS_HUMAN_REVIEW**: C-M01（组织 Intune 双层策略合并优先级——README 主张 vs 引擎无实现；需 Owner 或后续轮次在 host/ 目录验证）
- **Benchmark case（供 skill CI Gold Record）**:
  - BC-MXC-1: "策略指纹 = allow-list 投影 + 穷举 destructure + 凭据排除 + 残余披露文档化接受"——可审计且不泄露的策略身份基准样例
  - BC-MXC-2: "--die-with-parent = 时间线逃逸防御"——沙箱超时 kill 后的后代逃逸失败模式样例

## 结论
- 主 Package 核心结论全部 CONFIRMED（0 CONTRADICTED）
- 3 项 PARTIALLY_CONFIRMED / 2 项 OVER_GENERALIZED / 1 项 MISSING 已在 Reconciliation 阶段落地修正（见 06_validation/reconciliation.md）
- 审计修正不删除 EK（EK 不得因形成 KO/修正而消失）；全部修正可反向追溯独立证据
