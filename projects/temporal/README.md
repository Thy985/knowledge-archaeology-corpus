# Temporal — Knowledge Archaeology

## 一句话定位

Temporal Server（Go，go.temporal.io/server）——**durable execution 品类鼻祖，Replay-based（确定性重放）执行模型的代表实现**：持久化的不是"执行"而是"事件序列"，workflow 代码在恢复时重放、只对日志中尚不存在的事件产生副作用。与 Restate（journal 步骤持久化）、agent-substrate（挂起恢复）构成三大执行模型对照。

## 产物清单（ARCH-2026-11-02-001）

| 层 | 文件 | 说明 |
|----|------|------|
| 00 Overview | `00_overview.md` | TL;DR：命令→事件闭环、确定性重放、Chasm 双引擎、TeamMind 启示 |
| 01 Project Layer | `01-project-layer/project_layer.md` | 四服务架构 + 核心模块 + 数据结构 + 执行模型 + 测试/配置/治理 |
| 02 Engineering Knowledge | `02-engineering-knowledge/ek_graph.md` | **40 EK**（EK Graph 六类边，平均出边 2.1，游离 0，聚合覆盖率 100%） |
| 03 Knowledge Layer | `03-knowledge-layer/knowledge_layer.md` | **9 KO**（R1×3/R2×2/R3×2/R4×2，L3/L4；含反例限定） |
| 04 Flow Atlas | `04-flow-atlas/flow_atlas.md` | 七类流 38 Edge，symbol 可回溯 |
| 05 Candidates | `05-candidates/candidates.md` | 6 候选（3 Hypothesis + 3 Cross-project） |
| 06 Validation | `06-validation/validation_report.md` + `independent_validation_report.md` | Truth 8/8 + Flow 7/7 + 反例 9/9 + IVR（CONFIRMED×8 / PARTIALLY_CONFIRMED×5 / 0 CONTRADICTED） |
| Run 产物 | `archaeology-runs/ARCH-2026-11-02-001/` | run_metadata / job_manifest / snapshot_artifact |

## 考古元信息

- **run_id**：ARCH-2026-11-02-001
- **commit**：e0233ff6658fcab7b4e7ecc1c16af40baf629b78（2026-10-31，main，shallow clone depth=1）
- **skill_version**：knowledge-archaeology v3.2
- **证据上限**：S4（静态源码 + 测试揭示行为；无 Docker 运行验证，不升 S5）
- **规模**：1,124,761 行 Go（业务 1,023,714；Chasm 10.2 万行）；四服务（history 265,806 / matching 52,830 / worker 49,069 / frontend 39,997）

## Corpus 连接（跨项目对照）

- **Restate**（ARCH-2026-11-01-001，journal 步骤持久化）：replay vs journal 恢复哲学对照 → C-04 保留为 Cross-project Candidate
- **agent-substrate**（挂起恢复）：三大执行模型对照
- **Dapr Agents / dbos-transact**（durable 品类横向）
- **TeamMind**（用户 Java 多 Agent 项目）：Temporal Java SDK 直接选型候选

## 考古日期

2026-11-02
