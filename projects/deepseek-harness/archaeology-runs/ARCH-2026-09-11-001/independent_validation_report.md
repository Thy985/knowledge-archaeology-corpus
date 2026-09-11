# Independent Validation Report — ARCH-2026-09-11-001

> 方法：Auditor **未**以本 Archaeoology Package 为事实来源，独立重读仓库（`git ls-tree` 基线对比 + 代码 symbol 抽样 + notes 抽样 + pytest 实测），建立 Independent Findings 后再与本包对照。

## 一、Independent Findings（先于对照建立）

1. `packages/core/tools/src/index.ts`：`resolveExecution(name, scope, nested)`（L1211）+ `collapses(name, scope, nested)`（L1314，"security-relevant predicate"）+ `UNKNOWN_TOOL`（L326 注释"before the policy pipeline"）+ `TOOL_ABORTED_BEFORE_DISPATCH`（L465）+ nested 判定 `exec.parent !== undefined`（L1267）。PTC 折叠真实存在于执行边界。
2. `packages/core/session/src/surface.ts`：`surfaceOp` append/replace 判别（L45-75）+ 拒绝 `header.system`（L149）+ 拒绝空 `adapterDefaults`（L153-155）+ `isError === true` 强制（L163-164）。V3 envelope 校验真实存在。
3. `packages/api/workspace-files/src/`：Host（index.ts/changes.ts/types.ts）+ Client（client/）+ 文件头注释明确"file methods inherit the addressed session's read authority… outside the workspace… list and changes remain workspace-scoped"。
4. `readByteRange` 由 workspace-files / e2b / tool-cordis 三处实现（dsh-fs seam 多 provider）。
5. `python/sdk/tests`：**111 passed, 7 skipped**（独立实测，3.35s）。
6. `.agents/notes/implemented/{architecture,feature,bug-fix,process,simplification,testing}/` 958 条（git 实测 862→958）；基线 commit 76fda72 已存在 862 条。
7. 未发现绕过 PTC collapse 的直呼路径（resolveExecution 全仓唯一定义；nested 仅由 parent token 触发）。

## 二、对照判定

| 本包条目 | Auditor 判定 | 依据 |
|---|---|---|
| EK-R02 PTC 演化链 | **CONFIRMED** | notes 5 篇时间线 + 代码现状一致 |
| EK-R03 executor collapse | **CONFIRMED** | 代码 symbol 级（见上 #1） |
| EK-R04 workspace-files 服务 | **CONFIRMED** | 包结构 + Remote types + 头注释 |
| EK-R05 读权限分界 | **CONFIRMED** | index.ts 头注释逐字对应 |
| EK-R07 V3 envelope | **CONFIRMED** | surface.ts 校验函数级 |
| EK-R09 python SDK | **CONFIRMED** | pytest 实测（S4） |
| EK-R01 notes=ADR 证据源 | **PARTIALLY_CONFIRMED** | 958 条存在；"frozen archive"未逐一验证；双语配对抽查通过 |
| EK-R08 iframe 权衡 | **PARTIALLY_CONFIRMED** | notes 记录充分；client 侧 iframe 代码未逐行核（web 客户端包大） |
| EK-R06 改名纪律 / EK-R10 SAFETY / EK-R11 反馈 / EK-R12 subagent / EK-R13 持久化 | **CONFIRMED（notes 级）** | 对应 notes 文件存在且内容一致 |
| KO-09 呈现/执行双面强制 | **CONFIRMED（L4 恰当，未过升）** | collapse 反例（旧版）+ 合法例外（nested）均代码可证 |
| KO-10 操作语义分层权限 | **CONFIRMED（L4 恰当）** | read vs list/changes 真实分界 + 09-05/09-09 supersede 历史 |
| C-R05 "181 插件" | **CONFIRMED（标注一方称正确）** | 仓库仅 55 packages 可核验 |
| 09-03 基线 32 EK / 8 KO 存续 | **CONFIRMED（无反证）** | 本轮未发现 v0.1.5 反证基线主张的代码/notes |

**判定统计**：CONFIRMED 12 / PARTIALLY_CONFIRMED 2 / DOWNGRADED 0 / OVER_GENERALIZED 0 / MISSING 0 / CONTRADICTED 0 / NEEDS_HUMAN_REVIEW 0。

## 三、必须回答的 6 问

### 3 个最重要的成功
1. **PTC collapse 从"笔记声称"验证为"代码事实"**——resolveExecution/collapses/nested/UNKNOWN_TOOL 逐 symbol 对上，且确认折叠先于策略管线（L326 注释），这是 09-03 EK-28 的升级证据。
2. **发现并补偿 09-03 整块 Coverage Gap**——.agents/notes（862→958 条 ADR）在 09-03 包中零引用；本轮 13 条新 EK 全部以它为主证据源，回答了"为什么"层问题。
3. **雷达声明证据分级落地**——"文件上传/侧栏预览"在 notes 中查证；"181 插件/三模式联合训练"明确标为一方称，未冒充已查证。

### 3 个最重要的错误（本包自身）
1. **EK-R08（iframe）判定为 PARTIALLY_CONFIRMED**——Auditor 未在 client 包逐行核 iframe `sandbox` 属性，仅 notes 级确认；如需 S4 需补 client 代码证据。
2. **EK-R01 "frozen archive"表述过强**——Archived 机制存在（archived/ 目录），但"不可变"未做 git 历史验证，已降为 PARTIALLY_CONFIRMED。
3. **acp/gcal/schedule 等 09-03 登记 gaps 仅部分补偿**——feedback/code-runtime 已补，acp 桥仍浅，属登记而非解决（诚实标注）。

### 是否存在关键遗漏
- 无新发现的整块遗漏；但 web 前端（client 包）与 acp 协议桥仍是浅覆盖，登记为持续 gap。
- "181 插件生态"无仓库内核验途径，作为媒体声明保留。

### 是否存在错误升维
- 否。KO-09/KO-10 均为 L4 且标 Cross-project pending；无 L5；5 个新 Candidate 全部标 Hypothesis/Tentative/Scope-uncertain。

### 是否存在事实错误
- 否（本 Auditor 独立抽样未发现与仓库实际内容冲突的声明；9 个数字类声明如 notes 862/958、packages 55、111 tests 均 git/pytest 实测）。

### 是否存在 Flow 错误
- 否。Control（collapse 先于 createExecution）、State（V3 接受/拒绝）、Data（workspace-files 七方法 + readByteRange）、Authority（read vs list 分界）四条新增边全部对回真实 symbol/函数/注释。

### 是否发现新的 Benchmark / Regression Case
- **是，3 个可沉淀**：
  1. **PTC collapse 回归测试**：mode='ptc' + 模型直呼 native tool（无 parent）→ 必须 UNKNOWN_TOOL 且**不触发** pre-execute/approval/guard（策略管线零观察）。这是"schema 省略≠强制"的回归锚。
  2. **V3 envelope 接受规则测试**：`header.system` 存在 / 空 `adapterDefaults` / `data.error` 缺 `isError===true` → 必须拒绝。
  3. **workspace-files 权限分界测试**：read 族可越 workspace 读（后端允许时）；list/changes 必须 workspace-scoped 拒绝越界。

## 四、结论

**ACCEPT**。未修改原 Archaeology Result；上述 PARTIALLY_CONFIRMED 项（EK-R08/EK-R01）与 3 个 Benchmark Case 将进入 Reconciliation 标注，不改变验证结论。
