# Agent Substrate 知识考古索引

> 一句话定位：**零信任 agent 沙箱运行时**（Google Agent Substrate 开源实现，Go/K8s-native）——海量有状态 actor 映射到少量热 worker，空闲整机快照、请求到来恢复；每个 agent 有身份、无凭据、无开放网络、无法逃出沙箱。

## 考古信息

- 仓库：https://github.com/agent-substrate/substrate.git（commit `59a58a5f`，v0.6.0 2026-10-22，Apache-2.0，Go 1.27，1425 文件≈42.7 万行 / 670 测试文件）
- 考古日期：2026-10-26 · run_id：`ARCH-2026-10-26-001` · skill：knowledge-archaeology v3.2
- 来源：KnowlegeMap Runtime 周二号锚点（10-22 发布、10-24 fallback scan 标记 validated 门槛）

## 产物清单

| 目录/文件 | 内容 |
|---|---|
| `metadata.yaml` | 考古元数据（commit/run/summary/判定统计） |
| `00_overview.md` | 总览：六条认知主线 + 关键数字 + corpus 跨项目连接 |
| `01-project-layer/project_layer.md` | 项目地图：双存储体系/七态状态机/快照体系/测试/治理 |
| `02-engineering-knowledge/ek_graph.md` | 20 条 EK（EK Graph 六类边，平均出边 2.25，游离 0） |
| `03-knowledge-layer/knowledge_layer.md` | 7 个 KO（R1×1/R2×2/R3×2/R4×2 + 聚合规则矩阵） |
| `04-flow-atlas/flow_atlas.md` | 七类流 22 条 Edge（Control/State/Data/Evidence/Authority/Memory/Policy） |
| `05-candidates/candidates.md` | 7 候选（含跨项目假说 C-S05 凭据外注三仓收敛） |
| `06-validation/validation_report.md` | 六审计 + 质量指标 + 待 IVR 攻击面 |
| `06-validation/independent_validation_report.md` | 盲重建报告（11 CONFIRMED/2 PARTIAL/1 CONTRADICTED/4 MISSING/1 REVIEW） |
| `archaeology-runs/ARCH-2026-10-26-001/` | 本轮原始产物（job_manifest / snapshot_artifact / run_metadata / independent_validation_report） |

## 关键发现（Top 6）

1. **快照-恢复三件套**：分级快照（VOLUMES/ROOTFS/MEMORY 每级含下级）× 幂等 checkpoint（`.checkpoint-done.json` 完成标记，超时/重启重试直接复用）× 本地/远端双持久化（locality + durableSnapshot PENDING→COMPLETED）——与 dapr-agents 的 durable execution（步骤日志）构成"恢复"两种哲学
2. **零信任四支柱**：必沙箱（明文容器不支持）× 零环境凭据（凭据只在水合瞬间出现）× 默认拒绝网络（egress fail-closed PEP + ingress mTLS allowlist）× 组件 mTLS 全覆盖（1h SPIFFE X.509，唯一豁免 /healthz /readyz）
3. **身份代行可区分**：actoridjwt 双 Kind（`actor/` vs `worker-for-actor/`）——同一 actor 的自身代码与组件代行主体分离，依赖方可授不同权；ClaimsToWire 拒绝未设 Kind（防漏签发）
4. **请求停车**：worker 饱和时 router 挂起请求（lot 1024 + budget 5s + 指数退避），瞬时资源压力从客户端 503 转成内部重试；ext_proc 熔断器不得截断 lot（启动校验）
5. **愿望/实现分离制度化**：architecture.md 首行 "aspirational, and is not yet implemented!" + security.md Met/In-Progress/Future 状态表 + threat-model 拒绝固化状态（防 bias 自动化评估）——考古的审计锚
6. **legacy 路径发现（盲重建）**：`--upstream-connect=false` + 无 credential bundle 时 dial pod-IP:80 plaintext——"worker 80 非入口"仅默认配置成立（Reconciled 补正）

## Corpus 跨项目连接

- **dapr-agents（10-25）**：durable execution vs 整机快照——"恢复"两种哲学；审批/HITL vs 零信任网络策略——遏制机制对照
- **MXC / Strands Box（Harness 周）**：OS 隔离+凭据外挂 vs Substrate 零信任+注入后丢弃——凭据外注证据线三仓收敛（C-S05 假说）
- **DeepSeek Harness（validated）**：sandbox/approval/credentials seam vs 硬隔离默认拒绝——harness→runtime 强化对照
