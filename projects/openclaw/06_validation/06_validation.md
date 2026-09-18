# 06 Validation & Evidence — OpenClaw

> Validation 遵循 skill v3.2：Truth/Coverage/Flow/Abstraction/Counterexample/Epistemic 六项，先盲重建再对比。本文件为 Archaeologist 自检记录；阶段 5 独立 Auditor 报告另列。

## 6.1 Evidence Register（关键证据表）
| ID | 证据 | 位置 | 支撑 |
|---|---|---|---|
| E-01 | visibility 省略值 agent→all | docs/releases/2026.9.2.md:3245 | EK-01 |
| E-02 | agentToAgent 省略值 false→true | 同上 | EK-01 |
| E-03 | visibility 四档定义 + 默认 all | zod-schema.agent-runtime.ts:770-780 | EK-05 |
| E-04 | agentToAgent enabled 默认 true + allow 语义 | session-visibility.ts:244-276 | EK-04/28 |
| E-05 | invalid visibility→all | session-visibility.ts:148-161 + 测试 | EK-25 |
| E-06 | audit checkId + severity info/warn | audit-extra.summary.ts:241-251 | EK-02 |
| E-07 | 审计测试 7+4 表驱动 | audit-cross-agent-session-access.test.ts | EK-30 |
| E-08 | SECURITY.md "not security boundaries" | SECURITY.md | EK-07 |
| E-09 | swarm 默认配置有界 | swarm-config.ts:11-16 | EK-08 |
| E-10 | SwarmGroupLane + ALS | swarm-scheduler.ts:30-65 | EK-09 |
| E-11 | policy 先于 authority 发布 | tool-authority.runtime.ts:37-60 | EK-11 |
| E-12 | spawn 三阶段管线 | spawn-pipeline.ts:20-60 | EK-12 |
| E-13 | 权限模式四档 + full 需 admin | permission-modes.md | EK-16 |
| E-14 | control-plane owner-only | tool-permissions.md | EK-17 |
| E-15 | sandbox 不隐藏 transcript | docs/releases/2026.9.2.md:3249 | EK-24 |
| E-16 | memory agent-scoped vs sessions permission-scoped | 同上 | EK-23 |
| E-17 | tree/all 下 requester-owned child 仍可达 | schema.help.runtime.ts:117 | EK-03/04 |
| E-18 | 升级提示 + audit 命令 | docs/releases/2026.9.2.md:11,3251 | EK-27/37 |
| E-19 | Code Mode fail-closed | schema.help.runtime.ts:138-139 | EK-22 |
| E-20 | 原生 harness 自有工具面 | permission-modes.md | EK-38 |

## 6.2 Truth Audit（源真）
- 全部 38 条 EK 均有文件/行号/符号锚点（E-01~E-20 覆盖核心 20 条，其余见 EK 内联证据）。
- 文档事实与代码事实交叉验证：docs/releases 叙述（E-01/02/15/16/18）与实现（E-04/05/07）一致。
- 未发现"文档声称但代码不存在"的实质性矛盾。
- 未全量运行测试（15,643 个）；关键测试（audit-cross-agent-session-access）精读验证。

## 6.3 Coverage Audit（覆盖）
- 覆盖：权限模型变更（visibility/agentToAgent/swarm）、审计、信任模型、spawn 权威、approval、记忆、运行时选择。
- 未深入：channels 接入实现（20+ 渠道的具体鉴权）、plugin 生态细节、客户端 apps（8 个）、全部 22 packages。
- 判定：对本考古焦点（权限模型）覆盖充分；非焦点面按边界声明裁剪。

## 6.4 Flow Audit（流）
- 七类流均从真实代码导出，关键 Edge 可回溯 symbol/file（见 04 Flow Atlas）。
- 主动标注 5 个 Flow 缺陷候选（旁路/缺口）供阶段 5 攻击。

## 6.5 Abstraction Audit（抽象）
- L3 模式 8 个：均有跨项目对照或明确 Cross-project validation pending 标注。
- L4 模型 8 个：均为"一句话稳定关系"，未过度升维。
- 无"单案例→Pattern"的无据升维（C-01/C-08 明确标 Hypothesis）。
- 未把"项目经验"写成"普遍定律"（所有 generalization 带 pending 标注）。

## 6.6 Counterexample Audit（反例）
- 主动寻找 5 类反例：bypass（harness 旁路 C-07）、override（invalid visibility E-25）、exception（incognito EK-36）、alternate path（agentToAgent=false 后 child 可达 E-17）、fallback（fail-open 解析 E-25）。
- 每个 L3+ KO 均有 ≥1 个定向反例攻击点（详见 05 Candidates）。

## 6.7 Epistemic Audit（认知状态）
- Fact：E-01~E-20（源真验证）。
- Observation：EK-01 等（含 Why 解释的为 L2 工程知识）。
- Hypothesis：C-01~C-10（全部标 Hypothesis，不冒充知识）。
- Validated Pattern：KO-01~08（OpenClaw 内证据闭合；跨项目验证 pending 显式标注）。
- 无 Principle/Law 级升维（跨项目证据不足，主动降级）。

## 6.8 质量指标
| 指标 | 值 |
|---|---|
| Evidence 数 | 23（Reconciliation 后） |
| EK 数 | 41（Reconciliation 后；全部含 links，六类边，无游离） |
| KO 数 | 8（R1×1/R2×3/R3×2/R4×2） |
| Candidates | 10（全部 Hypothesis） |
| 七类流 | 7/7 |
| 本地测试运行 | 未全量（15,643 超出预算）；关键测试精读 |
| 已知局限 | 仓库 821MB；apps/packages/channels 深度未覆盖 |

## 6.9 Reconciliation（阶段 6，合并 Independent Validation）
独立 Auditor（独立重读仓库，未以本结果为事实来源）发现：
- **MISSING×4**：①子 agent 工具 deny-wins（agent-tools.policy.ts:26-42 SUBAGENT_TOOL_DENY_ALWAYS）→ 新增 **EK-39**；②ACP 通道 session 工具归控制面审批（approval-classifier.ts:26-32）→ 新增 **EK-40**；③删除 agent 后 allow 空→回退 allow-all（sessions-and-subagents.md:29）→ 新增 **EK-41**；④hardened-baseline 文档未引用 → 已补入 KO-03 remediation 建议。
- **OVER_GENERALIZED×2**：KO-01 补"主 agent 之间"限定（子 agent deny-wins 例外）；KO-04 补 allow 回退缝隙。
- **Flow 补边×2**：Authority Flow 补 ACP control_plane 分支 + 子 agent 收缩分支。
- **新增 Benchmark/Regression Case**：B-01（SUBAGENT deny 断言）、B-02（allow 回退语义）、B-03（ACP 分类基准）、B-04（audit severity 漂移边界）、B-05（invalid visibility fail-open fixture）。
- 未发现事实错误/Flow 错误；NEEDS_HUMAN_REVIEW 2 项（N-01 全量测试未运行、N-02 设计意图确认）。

## 6.10 交付物清单
- snapshot_artifact.md / 00_overview.md / 01_project-layer.md / 02_engineering-knowledge.md / 03_knowledge-layer.md / 04_flow-atlas.md / 05_candidates.md / 06_validation.md（本文件）
- run_metadata.yaml（另列）
