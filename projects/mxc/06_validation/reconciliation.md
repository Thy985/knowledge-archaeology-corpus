# Reconciliation — ARCH-2026-10-19-001（MXC）

> 合并 Independent Validation 修正（管道顺序/遗漏/scope 收紧）。只修正 Corpus Artifact，不修改生产 Skill。

## 修正清单

### R-1 · EK-M05 补充 kill_for_timeout 语义（修正 E-1）
- **修正内容**: 补充"默认实现 = `self.kill()`"；"超时终止的树范围是后端实现契约（Unix 进程组 / Windows job object / state-aware adapter 仅确认前台进程）——TimedOut 不代表树级 kill"
- **独立证据**: sandbox_process.rs:264-282、:286-290（wait 注释）
- **落地**: EK-M05 内容扩充（保留原 links）

### R-2 · EK-M03 时间限定（修正 E-2）
- **修正内容**: "后端选择唯一所在地"限定"当前版本 1.0.1"；补注 dispatch.rs:21 文档预告实验性后端 run-to-completion 将来也路由进引擎
- **落地**: EK-M03 内容扩充

### R-3 · 新增 EK-M21 · Bubblewrap 时间线逃逸防御（修正 E-3）
- **新增 EK**: `--die-with-parent` 的精确动机——`--unshare-pid` 单独不够：bwrap forks 致新命名空间 pid1 为子进程；无该标志时后台化后代可在超时 kill 后存活并继续运行在已拆除的网络强制之外（bwrap_command.rs:432-437 SECURITY 注释 + run_bwrap_network_proxy_test.sh 断言）
- **links**: mechanism↔EK-M06（kill_for_timeout 族）；causal→EK-M14（网络强制拆除后逃逸）
- **层**: L1/L2

### R-4 · EK-M07 补充"无 sink 零成本"（采纳 S-2）
- **修正内容**: log_audit_event 仅在诊断 sink 附着时写，否则廉价空操作（logger.rs:387-390）
- **落地**: EK-M07 内容扩充

### R-5 · EK-M08 补充解析器层硬编码（采纳 S-1）
- **修正内容**: 双层防御——引擎 require_experimental_optin gate + config_parser 硬编码 experimental_enabled=false（state_aware.rs:427-430）
- **落地**: EK-M08 内容扩充

### R-6 · EK-M17 补充日志注入防护（采纳 A-4）
- **修正内容**: UnsupportedVersion 错误明确要求用户输入转义后再进日志（version.rs:48-58）
- **落地**: EK-M17 内容扩充

### R-7 · C-M01 提升 NEEDS_HUMAN_REVIEW 标记
- **修正内容**: 组织 Intune 双层策略合并优先级——README 产品主张 vs 引擎无实现；明确标注 NEEDS_HUMAN_REVIEW（host/ 目录后续轮次验证）
- **落地**: C-M01 标注更新

## 未修改项
- KO-M01~M05（验证无攻击成立）；Flow Atlas（七类流 Edge 全部确认）；00/01/03/04/05 主体内容
- 原考古产物禁止修改原则：以上修正以"corpus 落盘版本"为准，本地 package 保留初版 + 本文件记录差异

## Reconciliation 后 EK 统计
- EK 总数: 21（M01~M21）；全部带 links；平均出边 ~1.6；游离 0%
