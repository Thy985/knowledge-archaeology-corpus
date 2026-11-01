# Box（Strands Box）— Knowledge Archaeology 索引

## 一句话定位
AWS 开源的 OS 级 AI agent 行为遏制沙箱（Rust/Apache-2.0）：OS 隔离（macOS Seatbelt 当前 / Linux namespace 代码在）+ Dogwood 策略引擎（Cedar 超集+时间性规则）对 Strands Shell/Monty/egress gateway/MCP broker 四类 enforcement point 逐请求判定。**"沙箱限能触达什么，策略限能对触达之物做什么；凭据永远不进 agent 进程。"**

## 产物清单
| 层 | 文件 | 内容 |
|---|---|---|
| Overview | 00_overview.md | 定位 / 认知主线 / 跨项目连接 |
| Project Layer | 01_project-layer/01_project_map.md | 架构 / 进程拓扑 / 生命周期 / 数据结构 / 测试 / 配置 / 治理 |
| Engineering Knowledge | 02_engineering-knowledge/ek-graph.md | EK-B01~B22（EK Graph 六类边） |
| Knowledge Layer | 03_knowledge-layer/ko.md | KO-B01~B05（aggregation_rule R1-R4） |
| Flow Atlas | 04_flow-atlas/flow-atlas.md | 七类流（Control/State/Data/Evidence/Authority/Memory/Policy） |
| Candidates | 05_candidates/candidates.md | C-B01~B07（Hypothesis 标注） |
| Validation | 06_validation/ | validation + independent-validation-report + reconciliation |
| Runs | archaeology-runs/ARCH-2026-10-20-001/ | job-manifest / snapshot / run_metadata |

## 考古记录
- **ARCH-2026-10-20-001**（initial, preview @ 069c39c）— 完整六阶段；Harness/治理 周 Day2
- 同生态：dogwood（策略语言本体）09-08 考古；本考古为宿主实现

## 关键发现（3 条）
1. **授权面唯一化**：`CapabilityOutcome{Applied,Unavailable}` 无 Allow/Deny——机制不可能授权；deny-only floors 只能减
2. **凭据 = 注入点的配置**：agent 持占位符，gateway 边界换真（env:// / aws:// 等五源）；"credential binding is not authorization"
3. **时间性策略 = 跨调用因果**：单引擎共享历史让 fs:read → 限时拒 net 可表达；加载期拒绝惰性规则防策略静默失效

## 关键争议
- C-B01（组织级多 box 凭据隔离）NEEDS_HUMAN_REVIEW
- EK-B08 文档-实现漂移：credentials.md"两 scheme" vs 实现五源（Reconciliation R-1）
