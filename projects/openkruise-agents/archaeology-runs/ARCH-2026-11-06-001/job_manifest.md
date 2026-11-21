# Job Manifest — 2026-11-06（ARCH-2026-11-06-001）

> 输入：KnowlegeMap @ `750046e`（radar watch 2026-11-05）。Runtime 周第二周（11-02~11-08），今日为第二周 Day5（周五前一天增量日）。
> 排序维度：Knowledge Value / Agent-AI Engineering 相关性 / Novelty / 增长信号 / 近期发布 / 与已有 Corpus 连接 / 是否已考古 / 是否需要 refresh / Benchmark 学习价值（**不按 star 排名**）。

## 1. 候选状态盘点

| 候选 | 状态 | 依据 |
|---|---|---|
| Dapr Agents | COMPLETED | corpus `projects/dapr-agents/` |
| duraре / Temporal / Restate / DBOS / Dapr / Infinity / SemanticRun | COMPLETED / IN_PROGRESS(PR) | corpus 已有目录（semanticrun 为 11-05 PR #65 待合入） |
| Agent Substrate | COMPLETED（知识考古）/ REFRESH_REQUIRED（11-07 复现验证） | corpus `projects/substrate/` 已存在；KnowlegeMap 11-07 计划为**复现验证**，非知识考古 |
| E2B / Box / Strands Box | COMPLETED | corpus 已有目录 |
| **OpenKruise Agents**（`openkruise/agents`） | **NEW** | 11-03 CNCF 官方 blog 首次进入雷达；corpus 无 openkruise/kruise 目录 |
| Genspark SB-Box | QUEUED（不可考古） | 商业闭源，第三方单源报道，无源码 |
| AWS AgentCore maxLifetime | QUEUED（不可考古） | 平台闭源，仅操作文 |
| Agent formal verification / Tofu MCP fuzzing 等候选卡 | QUEUED | 非 Runtime 周主线条，留待后续领域 |

## 2. 候选排序（本轮）

| 排名 | 候选 | 知识价值 | Agent-AI 相关性 | Novelty | 增长信号 | Corpus 连接 | 已考古 | Benchmark 价值 | 结论 |
|---|---|---|---|---|---|---|---|---|---|
| **1** | **OpenKruise Agents v0.6.0** | 高：K8s 原生 sandbox lifecycle operator（CheckpointRestore / PoolAutoscaler / probe 驱动 pause-resume） | 极高：Agent Sandbox 基础设施层 | 中高：v0.6.0（2026-10-09）+ CNCF blog 11-03 | 强：pushed 2026-11-05、157 forks、310 open issues | 强：e2b/box/strands-box（沙箱）+ durable 家族（生命周期语义） | **否** | 高：checkpoint-restore 语义、pool 水位、wake-on-traffic | **选中** |
| 2 | Agent Substrate refresh | 高 | 极高 | 低（已考古） | 中 | 强 | 已考古 | 高 | 11-07 复现验证，本轮不做知识考古 |
| 3 | AgentCore maxLifetime / Genspark SB-Box | 中 | 高 | 中 | 中 | 中 | 不适用（闭源） | 低 | 不可深度考古（无源码） |
| 4 | 其他 Runtime 新动态（DBOS 1M wf/s 等） | 中 | 中 | 低 | 弱 | 强 | 已考古（增量并入品类卡） | 低 | 仅品类卡增量，不单独立项 |

## 3. 选中项目

| 字段 | 值 |
|---|---|
| job_id | `ARCH-2026-11-06-001` |
| project | OpenKruise Agents |
| repository | https://github.com/openkruise/agents.git |
| mode | initial |
| priority | A |
| source_entry | `03_expansion_queue/candidates/[cand]sandbox-lifecycle-suspend-resume-2026-11.md`（2026-11-05 新卡，B 级）；CNCF blog 2026-11-03 |

### 选中理由
1. **流派空白**：corpus 沙箱族（e2b/box/strands-box）均为单机/云端 SDK 或 OS 容器，缺少「K8s 原生 CRD + operator 管理 agent sandbox 生命周期」这一层；OpenKruise Agents 补上「编排层生命周期」空白。
2. **与 11-07 验证日直接对照**：其 CheckpointRestore / auto pause-resume 是 Agent Substrate `<500ms suspend-resume` 基准的 K8s 侧实现参照，本日考古可为明日复现验证提供工程事实底座（KnowlegeMap 已将其列为对照素材）。
3. **工程可考古性**：Go operator（CRD/webhook/controller/reconciler），7.2MB 仓库、Apache-2.0、活跃提交（2026-11-05），具备完整控制流/状态流/权限流证据。
4. **Novelty 与增长信号**：v0.6.0 于 2026-10-09 发布，11-03 CNCF 官方 blog 推介；157 fork / 310 open issues 显示真实使用面。
5. **不按 star 排序**：315 star 低于多数已考古项目，但知识增量（K8s 生命周期编排）高于同周其他候选。

### 排除理由（关键）
- **Agent Substrate**：corpus 已完成知识考古；11-07 是**复现验证**任务（KnowlegeMap 的验证闭环），不属于本 SOP 的深度考古动作；今日重复考古会违反「每日最多 1 个深度项目 + 不覆盖历史」。
- **Genspark SB-Box / AgentCore**：闭源，无仓库可考古，只能作为对照证据（已在候选卡中记录）。

## 4. 预期产出
- Repository Snapshot Artifact（commit SHA / 版本 / 模块地图 / 依赖 / 测试 / 配置 / 权限治理）
- Project Archaeology Package 00-06 + run_metadata.yaml（EK Graph / KO 聚合 / 七类流）
- Independent Validation Report（盲重建）
- Corpus PR：`projects/openkruise-agents/`
