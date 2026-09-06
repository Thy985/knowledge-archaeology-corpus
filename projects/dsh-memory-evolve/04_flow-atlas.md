# 04 Flow Atlas — 七类流（从真实代码导出）

> lib/ 全部可读——本包 Flow 的 Edge 均可回溯到符号/文件/条件（与 evolver 考古的混淆盲区形成对比）。

## F1 Control Flow（控制流）

```
DSH host（cordis.patch.yml bundle 注册 lib/index.js）
  ├─ systemPrompt 注入（五轨记忆快照 → user-role tail message，仅在文本变化时重追加）
  ├─ tools 注册（memory/skill_manage/todo/de_session/…）
  ├─ commands 注册（/memory_review /todo …）
  ├─ agent/turn-stopping（回合计数 → reviewInterval 到 → 审查 DUE）
  ├─ tools/pre-execute + post-execute（ws-coord 写前检测/写后警告注入）
  ├─ sync-worker.mjs（独立进程：网络请求才走这里——同步命令异步化）
  └─ update（tag 检查 → dirty 拦截 → checkout → restartRequired）
```

**关键边证据**：lib/index.js（2265 行可读）；review.js（turn-stopping）；ws-coord.js（pre/post-execute）；sync-worker.mjs。

## F2 State Flow（状态流）

```
会话状态（DSH 原生）→ 插件状态：
  turn counter（agent/turn-stopping，不自动重置）→ DUE → memory_review_status complete → 重置
  记忆文件状态：五轨（USER.md/MEMORY.md/项目.md/日志/每日）→ 漂移守卫往返验证 → .bak.<ts>
  学习轨道：SUGGESTIONS.jsonl（建议）→ /memory_review 确认 → user track 转正
  技能状态：skills/<name>/SKILL.md（kebab-case）→ read 事件证据 → patch
  同步状态：身份键 → decideModeB（shared/legacy/shared-fresh/专属）→ 三路合并 → push/pull
  ws-coord 锁状态：declared（软）+ observed（硬）→ TTL 1-5min 续期 → release/过期
  更新状态：localTag/outdated/restartRequired/backoff(30min)/.corrupt-*
```

**关键边证据**：review.js（counter 不重置）；store.js（漂移守卫）；sync/repo.js（decideModeB）；ws-coord.js（TTL）；update.js（退避）。

## F3 Data Flow（数据流）

```
对话（DSH 会话）→ [回合结束] 自动写项目日志/每日日志（含 [git main] 标记）
  → [显式] memory 工具调用 → user track 直接写（确认门）
  → [审查] review 提议 → SUGGESTIONS.jsonl → 用户确认 → user track
  → [情绪] 【反馈】行（情绪/任务分类/原话）→ 日志
  → [技能] skill_manage create/patch → ~/.agents/skills/<name>/SKILL.md
  → [派单] codex/grok/hermes/kimi CLI 技能 → 外部 AI 输出 → 主会话汇总
  → [同步] filesets 按轨收集 → sync-worker 进程 → git push/pull → 三路合并 → 落盘
  → [上下文] 记忆快照 → systemPrompt tail（变化才重追加，保缓存）
```

**关键边证据**：lib/index.js（快照注入语义）；lib/sync/filesets.js；lib/sync/worker.js；skills/ 目录。

## F4 Evidence Flow（证据流）

```
read-before-write 证据：会话日志 tool/call 事件（skill_manage action=read <name>）→ patch 放行
  （无 read 证据 → 拒绝——AI 只能编辑自己读过的技能）
漂移守卫证据：磁盘 round-trip 解析结果 → 重写放行/拒绝 + .bak 备份
  （add 追加例外：跳过守卫但拒绝"存在且空"）
评审证据：note（提取）→ guard.accept（NFKC 归一化 → 空泛抑制 → 每轮一条 → 去重升级）→ delivery
  （FIFO 4096 历史——已接受 note 抑制，升级 nit→concern→blocker 放行）
同步证据：PROVENANCE 归属标记 → decideModeB 决策依据
  （main 无归属 → 保守专属分支；tag 信任校验：tag 须在 origin/main 历史）
```

**关键边证据**：lib/skills.js（read 事件检查）；lib/store.js（round-trip）；lib/advisor/guard.js（全规则）；tests/update.test.js（信任校验断言）。

## F5 Authority Flow（权威流）

```
用户（最高权威）：/memory_review 确认建议转正；采纳/拒绝待办；/memory 显式写入
  → AI 写入权限：user track 仅显式动作或确认后；auto 模式（reviewEnabled）全局记忆直写（主会话）
  → 插件权威：只做节拍器与写路径（不产生记忆内容）；独立子模块独立开关
  → 外部 AI（CLI 派单）：受托执行重活，输出回主会话（不直接写记忆）
  → DSH host：public seams 唯一入口（systemPrompt/tools/commands/subagents/approval）
  → 同步远端：git 仓库（专属分支/共享仓库）——身份键归属决定分支，绝不碰他人 main
```

**关键边证据**：lib/index.js（只走 public seams 注释）；README 场景四/五；sync/identity.js + repo.js。

## F6 Memory Flow（记忆流）

```
短期：会话上下文（DSH）+ 审查 turn counter
长期：
  用户档案 USER.md（每回合注入）
  全局事实 MEMORY.md（审查建议/确认）
  项目关键记忆 <project>/（git 分支生效标记；可归档/转回）
  项目日志/每日日志（自动记录，按需读取）
  学习轨道 SUGGESTIONS.jsonl（待确认）
  待办 TodoStore（四轨：生活/工作/项目/每日）
跨设备：sync（按轨开关 → 专属分支/共享仓库 → 拉取合并）——未开同步项目纯本地
回忆：recallVerifier 类机制 + 本地文件搜索（文件名/内容检索）
```

**关键边证据**：README 场景一（五轨）；store.js（五轨实现）；sync/*；search-docs.js。

## F7 Policy Flow（策略流）

```
决策（配置）：
  reviewEnabled/reviewInterval（审查节奏）
  同步轨开关（全局/每日/项目/待办）
  ws-coord.enforceWrite（软→硬）
→ 策略固化（执行点）：
  * 确认门：建议 → SUGGESTIONS.jsonl → 用户确认 → user track（写入策略）
  * 证据门：read-before-write（技能编辑策略）
  * 漂移守卫：round-trip 重写验证（文件保护策略）
  * decideModeB：PROVENANCE 归属 + 绝不碰 main（同步治理策略）
  * guard 闸门：空泛抑制 + 每轮一条 + 升级放行（评审输出策略）
  * 更新安全：dirty 拦截/退避/信任校验/API 同源（更新策略）
→ 未来决策反馈：
  * 情绪反馈积累（【反馈】行）→ 用户不满任务类型分析
  * 学习轨道 → 确认后的记忆转正
  * 用户拍板纪律注释（2026-08-08/08-09 决策内嵌代码——模块组织/锁 TTL 的历史决策可追溯）
```

**关键边证据**：README 配置段；ws-coord.js（enforceWrite 开关位）；guard.js（策略规则）；update.js（安全策略）；模块头部"用户拍板"注释（决策溯源）。

---

## 流完整性检查（Flow→KO 交叉校验）

| 流 | 对应 KO | 关键边可回溯 |
|----|---------|-------------|
| F1 Control | KO-01/KO-05 | ✅ turn-stopping + pre/post-execute |
| F2 State | KO-01/KO-03 | ✅ 计数器 + 合并状态 |
| F3 Data | KO-01/KO-05 | ✅ 五轨 + 派单 |
| F4 Evidence | KO-02/KO-04 | ✅ read 事件 + guard |
| F5 Authority | KO-01/KO-06 | ✅ 确认门 + public seams |
| F6 Memory | KO-01/KO-03 | ✅ 五轨 + 同步 |
| F7 Policy | KO-04/KO-05 | ✅ 策略执行点 + 用户拍板溯源 |

**可信度声明**：本包七类流全部来自可读实现 + 实测测试（798 断言），无混淆盲区——与同日 evolver 考古（57/202 混淆）形成可信度对照。
