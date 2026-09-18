# Independent Validation Report — letta-code（ARCH-2026-09-18-001）

> 独立 Auditor 盲重建：**不把考古结果当事实来源**，独立重读仓库（HEAD `3df2ebd6`）建立 Independent Findings 后对比。本报告不修改原考古产物。

## 盲重建方法
1. 独立重读记忆/治理核心路径：memory-confinement.ts / sandbox-policy.ts / bwrap.ts / seatbelt.ts / memfs-git-proxy.ts / mod-engine.ts / mod-sources.ts / memory-git.ts（pushMemory/getMemoryGitStatus）
2. 建立独立事实集（IF-01~IF-12）
3. 与原考古 Package 对比判定

## Independent Findings（盲重建事实集）

| IF | 内容 | 证据 |
|---|---|---|
| IF-01 | memory-confinement 顶层模块仅 3 个 import，直接委托 createMemoryConfinementLauncherWithAvailability + detectSandboxBackend | src/memory-confinement.ts |
| IF-02 | **canonicalizeRoot 用 realpath 解析符号链接**：注释明示"lexical path that passes through a symlink would silently match nothing — i.e. a sandbox that allows everything"；对不存在的 leaf realpath 最近存在祖先 | src/permissions/sandbox-policy.ts canonicalizeRoot |
| IF-03 | **bwrap 用 --tmpfs mask denied roots**：其他 agent 目录"not merely unwritable but *absent* — unreadable and unenumerable, strictly stronger than the static guard" | src/sandbox/bwrap.ts |
| IF-04 | **bwrap ancestor carve-out HAZARD**：carve-out 必须是 denied root 的后代或不相交，绝不能是祖先（last-mount-wins 会重新暴露 denied roots）；解释为何 memory-subagent profile 不 carve temp dir | src/sandbox/bwrap.ts 注释 |
| IF-05 | **seatbelt 用 (allow default) + 定向 deny**：威胁模型 = filesystem scoping for memory isolation，非通用不可信代码隔离（比 Codex (deny default) 更窄）；`/usr/bin/sandbox-exec` 硬编码防 PATH 植入 | src/sandbox/seatbelt.ts |
| IF-06 | **pushMemory 直接 `git push -u origin main`**；MemoryPostTurnSyncStatus = clean/pushed/dirty/conflict/push_failed/skipped——post-turn push 失败有显式状态记录 | src/agent/memory-git.ts pushMemory |
| IF-07 | **getMemfsServerUrl 显式忽略 LETTA_BASE_URL**：Desktop 的 ephemeral localhost proxy 由 LETTA_MEMFS_GIT_PROXY_BASE_URL 单独处理，git config 只持久化 canonical URL；proxy 用 url.<prefix>.insteadOf | src/backend/api/memfs-git-proxy.ts |
| IF-08 | **mods 全部 trusted: true**（legacy_global/global/agent 三源）；mod-engine 用 `await import(...?mod=mtimeMs)` 动态加载，createRequire 来自 runtime；无可见 mod 沙箱 | src/mods/mod-sources.ts / mod-engine.ts |
| IF-09 | memory 工具 reason 必填确认 | src/tools/impl/memory.ts:103 |
| IF-10 | 记忆子代理默认内核沙箱 + LETTA_FS_SANDBOX=0 opt-out 确认 | src/agent/subagents/sandbox.ts |
| IF-11 | FsSandboxPolicy 顺序语义 + baseWritableRoots 先于 deniedRoots 确认 | src/sandbox/policy.ts |
| IF-12 | 记忆写同步模式由 backend.capabilities.localMemfs 决定确认 | src/tools/impl/memory.ts getMemoryWriteSyncMode |

## 判定汇总

### CONFIRMED（31）
| 原 Claim | 判定理由 |
|---|---|
| EK-01 记忆=git 仓库 | IF-01 + memory-git.ts 全文确认 |
| EK-02 git 生命周期（clone/pull/commit/push） | pushMemory/getMemoryGitStatus 确认；post-turn push 状态机存在 |
| EK-05 系统提示词投影（{CORE_MEMORY}） | injectCoreMemory/renderMemfsProjection 确认 |
| EK-06 记忆写工具强制 reason | IF-09 确认 |
| EK-12 跨 agent 记忆墙（双树 deny） | getCrossBackendAgentsTreeRoots 确认 |
| EK-13 memory confinement fail-closed | IF-01 确认（无沙箱抛错） |
| EK-14 记忆子代理默认沙箱 | IF-10 确认 |
| EK-15 沙箱可用性探测缓存 | detectSandboxBackend cached 确认 |
| EK-17 FsSandboxPolicy 顺序语义 | IF-11 确认 |
| KO-01/02/04/05/07 主断言 | 盲重建路径全部命中 |

### PARTIALLY_CONFIRMED（9）
| 原 Claim | 缺口 |
|---|---|
| EK-03 pre-commit hook 全量规则 | hook 脚本确认，但"read_only 保护仅 legacy"的 server 侧语义未闭环（需 server 端证据） |
| EK-04 v2 白名单 | 本地校验确认；server 如何生成 v2 文件未验证 |
| EK-10 记忆迁移 skill 实际行为 | skill 存在确认，迁移逻辑未跑通验证 |
| EK-27 reflection 子代理 | 预算常量确认；reflection 实际触发→merge 全流程未盲测 |
| EK-39 skills 四源优先级 | 代码确认；实际优先级冲突行为未测试验证 |
| EK-46 worktree 固定身份 | 实现确认；"避免污染用户 git 身份"动机为推断 |
| KO-03 路径模型 | 路径墙确认；但见 DOWNGRADED-1 |
| KO-06 双运行时一致性 | AGENTS.md 声明确认；实际双路径测试覆盖度未全量核验 |
| KO-09 决策留痕 | reason→commit 链确认；reason 真实性未验证 |

### DOWNGRADED（2）
| 原 Claim | 降级理由 | 建议 |
|---|---|---|
| C-02（"符号链接逃逸未被验证"作为风险假设） | **IF-02 反证**：canonicalizeRoot 显式 realpath 防 symlink 逃逸，且注释点明漏洞动机。该假设从"scope-uncertain 风险"降级为"设计已防护、仅 bwrap/seatbelt 具体实现仍需审阅" | 从 Candidates 移除或改写为"realpath 防护已存在，剩余风险为内核层绕过" |
| C-05（"post-turn push 失败路径未闭环"） | **IF-06 反证**：MemoryPostTurnSyncStatus 含 conflict/push_failed/skipped 状态机——失败路径有显式记录；但补偿/重试策略仍未验证 | 改写为"失败有状态记录；补偿策略（conflict 恢复）未验证" |

### OVER_GENERALIZED（3）
| 原 Claim | 问题 | 建议 |
|---|---|---|
| KO-08 方法论"必须"断言 | 单项目导出即用"必须"措辞，虽有 pending 标注，但 L5 断言过强 | 措辞改为"建议准则"；保持 pending |
| KO-01 "记忆获得与代码同等的审计性" | "同等"绝对化；git 记忆审计 ≠ 代码审计（无 review/CI 对应物） | 改为"获得与代码同构的版本化审计" |
| EK-03 "写入门禁" | pre-commit hook 是本地脚本，非强门禁（git --no-verify 可绕过） | 标注"软门禁（可被 --no-verify 绕过），内嵌脚本设计意图是隔离环境自校验" |

### MISSING（2）
| 缺失项 | 证据 | 影响 |
|---|---|---|
| **bwrap ancestor-carve HAZARD 语义**（IF-04） | bwrap.ts 注释：carve-out 绝不可为 denied root 祖先 | 原 EK-17 只讲顺序，未讲 last-mount-wins 的祖先约束——补充后沙箱顺序语义更完整 |
| **seatbelt allow-default 威胁模型**（IF-05） | seatbelt.ts：允许默认 + 定向 deny，比 Codex (deny default) 窄 | 原考古"fail-closed"叙事需精确化：fail-closed 由 restrictWrites 的全局 deny 体现，profile 基底是 allow-default |

### CONTRADICTED（0）
盲重建未发现与原考古产物直接矛盾的事实 claim。

### NEEDS_HUMAN_REVIEW（1）
| 项 | 原因 |
|---|---|
| mods 全 trusted + 主进程动态加载（IF-08） | mods 与记忆子代理的 fail-closed 沙箱形成鲜明对比——用户需确认这是有意设计（可信插件模型）还是待加固面；原考古 C-08 保留但需人工裁决 |

## 重点攻击专项回答

### 1. 单案例→Pattern 攻击
- KO-01/03/04/05/06/09 均存在 ≥2 个独立子系统证据（记忆/权限/工具/CI），非单案例。
- **风险点**：KO-01 的"文件树记忆"跨项目证据（OKF/Grok/Anthropic）全部来自雷达扫描（二手信号），非本项目代码——已正确留在 C-01，未升 KO。✅

### 2. Pattern→L4 攻击
- KO-02（信任基线由回退路径决定）：盲重建确认双基线设计（subagents/sandbox.ts 注释直述"no approve/deny flow to fall back on"）。L4 升维有据。✅
- KO-07（决策轨/执行轨正交）：四模式 + 沙箱 + mods 能力面三条轨存在，L4 有据。✅

### 3. 项目经验→通用 Principle 攻击
- KO-08 已标 pending，不冒充普适。⚠️ 措辞"必须"过强（见 OVER_GENERALIZED）。

### 4. ADR→实现事实攻击
- AGENTS.md 规则（@/、kebab-case、双运行时）均有实现（check-boundaries.js / check-filename-casing.js / test 双路径）。✅
- AI_POLICY（贡献治理）是仓库政策，非代码实现——原考古已归 Policy Flow，未当实现事实。✅

### 5. Flow Edge 真实性攻击（逐条）
- Control Flow：index.ts → context → backend-mode → injectCoreMemory → tools/manager → memory 工具 → post-turn push。全部符号可回溯。✅
- Data Flow：**collectCommittedMemoryFiles 只读 HEAD 提交内容**（git ls-tree HEAD + git show）——关键 edge 真实且重要（投影一致性由 git 提交保证）。✅
- Memory Flow：初始化→写入→hook→commit→push→投影→reflection→迁移。全部符号可回溯。✅
- Policy Flow：决策→批准→策略→执行→反馈闭环中，"未来决策"环节无显式 RFC 流程证据（原考古已注明）。✅

### 6. 反例/旁路攻击（bypass/override/exception/alternate/direct call/admin path/fallback/legacy path）
| 路径 | 发现 | 处置 |
|---|---|---|
| bypass | `LETTA_FS_SANDBOX=0`（记忆子代理沙箱退出） | KO-02 反例已收录 ✅ |
| bypass | `git --no-verify` 可绕 pre-commit hook | OVER_GENERALIZED-3 新增 ✅ |
| override | `setConfiguredBackendMode` 覆盖 env 标志 | backend-mode.ts 单源真值确认 ✅ |
| fallback | getMemfsServerUrl fallback LETTA_CLOUD_API_URL；忽略 LETTA_BASE_URL | IF-07 补充（原考古未详述）✅ |
| legacy path | memfs-v1 / legacy frontmatter（read_only/limit）/ legacy permissions 字符串映射 | EK-04/EK-18 已覆盖 ✅ |
| alternate | "直接编辑投影文件自行 commit"（v2 同步指令） | EK-07 已覆盖 ✅ |
| admin path | mods 动态加载（createRequire + await import）无沙箱 | NEEDS_HUMAN_REVIEW ✅ |
| direct call | pushMemory 直接 `git push -u origin main` | IF-06 补充 ✅ |

### 7. Epistemic 状态混淆检查
- 原考古 KO 全部有明确 Epistemic 标注（Pattern/Model/Methodology+single-source）。✅
- C-01~C-08 全部 Hypothesis/候选，未混入 KO。✅
- 无 Fact 写成 Observation、无 Pattern 写成 Principle。✅

## 原 Archaeology 最重要的 3 个成功
1. **KO-02（fail-closed 治理）与 KO-07（治理双轨）的升维**：准确捕捉了 letta-code 最独特的工程认知——"无审批回退路径必须内核强制"与"决策轨/执行轨正交"；盲重建完全确认，且 subagents/sandbox.ts 注释提供了直接佐证。
2. **Data Flow 的"投影只读已提交内容"发现**：collectCommittedMemoryFiles 用 git ls-tree/show 读 HEAD——这是记忆一致性的核心设计意图，原考古抓住了。
3. **宽底座（46 EK + 64 边）**：记忆子系统 12 条 EK + 治理 14 条 EK 覆盖充分，EK Graph 无游离节点，聚合规则 100% 覆盖，符合 v3.1 契约。

## 原 Archaeology 最重要的 3 个错误
1. **C-02 符号链接逃逸假设错误**：原考古把"symlink 逃逸未验证"列为风险假设，但 sandbox-policy.ts 的 canonicalizeRoot 显式 realpath 防护（且注释明示漏洞动机）。这是盲重建最直接的纠错——**假设未做符号链接搜索**。
2. **C-05 push 失败"未闭环"表述过强**：MemoryPostTurnSyncStatus 状态机（conflict/push_failed/skipped）已存在，原考古未读到；"未闭环"应精确为"补偿策略未验证"。
3. **沙箱叙事精确性不足**：原考古统一用"fail-closed"叙事，但 seatbelt 基底是 allow-default + 定向 deny（威胁模型=FS 隔离）；fail-closed 仅由 restrictWrites 的全局写拒绝体现。bwrap 的 ancestor-carve HAZARD 也未在 EK-17 中说明。

## 关键遗漏
1. **bwrap ancestor-carve HAZARD**（carve-out 不可为 denied root 祖先；last-mount-wins）——补充到 EK-17 可显著提升沙箱语义完整性。
2. **seatbelt allow-default 威胁模型 + /usr/bin/sandbox-exec 硬编码防 PATH 植入**。
3. **mods 全 trusted + 主进程动态加载**——原考古 C-08 只是"假设全开默认面宽"，实际证据更强（三源 trusted:true + await import 无沙箱）。
4. **canonicalizeRoot realpath 防 symlink**（对应 C-02 纠错）。
5. **memfs-git-proxy 的 LETTA_BASE_URL 忽略逻辑**（Desktop ephemeral 与 canonical URL 分离）。

## 错误升维？
无错误升维。KO-02/KO-07 的 L4 有据；KO-08 正确标注 pending。草案中 2 项过强断言已在定稿前降级（记忆终局→C-01；fail-closed 唯一正确→对照叙事）。

## 事实错误？
盲重建未发现事实错误（CONTRADICTED=0）。唯一接近的是 C-02 假设（方向性错误，已在 DOWNGRADED 处理）。

## Flow 错误？
无。七类流关键 Edge 全部可回溯。

## 新 Benchmark / Regression Case 建议
| Case | 验证点 | 来源 |
|---|---|---|
| **symlink 逃逸防护回归** | 在自我 writableRoots 内建 symlink 指向 deniedRoots，断言沙箱仍拒绝 | IF-02 canonicalizeRoot |
| **ancestor-carve 拒绝回归** | 构造 carve-out = deniedRoot 祖先，断言策略构建拒绝或沙箱仍隔离 | IF-04 bwrap HAZARD |
| **post-turn push conflict 状态回归** | 远端推进后本地 push，断言 MemoryPostTurnSyncStatus=conflict 且提示 fetch first | IF-06 + NON_FAST_FORWARD 正则 |
| **LETTA_BASE_URL 不污染 git config 回归** | 设 ephemeral LETTA_BASE_URL，断言 memfs git remote 仍用 canonical URL | IF-07 getMemfsServerUrl |
| **read_only legacy 保护回归** | legacy 文件 HEAD read_only:true 时写操作被拒；v2 同字段不被识别为只读 | memory-frontmatter.ts legacyValue |
| **memory 工具 reason 空串拒绝回归** | reason="" 或缺失时工具抛错 | tools/impl/memory.ts:103 |

## 判定统计
| 判定 | 数量 |
|---|---|
| CONFIRMED | 31 |
| PARTIALLY_CONFIRMED | 9 |
| DOWNGRADED | 2 |
| OVER_GENERALIZED | 3 |
| MISSING | 2（并入 Reconciliation） |
| CONTRADICTED | 0 |
| NEEDS_HUMAN_REVIEW | 1（mods trusted 模型） |
| **合计** | **48** |

## Reconciliation 建议（供阶段 6）
1. 从 Candidates 移除 C-02（符号链接）或改写为"realpath 防护存在，剩余内核层风险"；C-05 改写为"失败有状态机；补偿策略未验证"。
2. EK-17 补充：bwrap last-mount-wins + ancestor-carve HAZARD；seatbelt allow-default 威胁模型。
3. EK-03 标注"软门禁（--no-verify 可绕）"。
4. C-08 升级证据：mods 三源 trusted:true + 主进程动态加载（保留 NEEDS_HUMAN_REVIEW）。
5. KO-01/KO-08 措辞收紧（审计性"同等"→"同构"；"必须"→"建议准则"）。
6. 新增 6 条 Regression Case 建议（见上）。
