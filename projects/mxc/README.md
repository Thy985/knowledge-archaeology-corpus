# MXC — Knowledge Archaeology 索引

## 一句话定位
Microsoft eXecution Container（MXC）：策略驱动的跨平台沙箱代码执行系统（Rust/MIT/GA 1.0.1）。应用把不可信代码交给 SDK，按统一 JSON 策略在 OS 原生隔离后端（Bubblewrap/AppContainer/Seatbelt/LXC/WSLC/IsolationSession/MicroVM）执行——**"agent 不能成为自身的安全权威"**。

## 产物清单
| 层 | 文件 | 内容 |
|---|---|---|
| Overview | 00_overview.md | 定位 / 核心认知 / 关键文件 |
| Project Layer | 01_project-layer/01_project_map.md | 架构 / 模块 / 数据结构 / 生命周期 / 测试 / 配置 / 治理 |
| Engineering Knowledge | 02_engineering-knowledge/ek-graph.md | EK-M01~M21（EK Graph，六类边） |
| Knowledge Layer | 03_knowledge-layer/ko.md | KO-M01~M05（aggregation_rule R1-R4） |
| Flow Atlas | 04_flow-atlas/flow-atlas.md | 七类流（Control/State/Data/Evidence/Authority/Memory/Policy） |
| Candidates | 05_candidates/candidates.md | C-M01~M07（Hypothesis/Observation 严格标注） |
| Validation | 06_validation/ | validation + independent-validation-report + reconciliation |
| Runs | archaeology-runs/ARCH-2026-10-19-001/ | job-manifest / snapshot / run_metadata |

## 考古记录
- **ARCH-2026-10-19-001**（initial, 1.0.1 @ fda29ac）— 完整六阶段；质量指标见 run_metadata.yaml
- 每日 SOP 轮：2026-10-19（Harness/治理 周 Day1）

## 关键发现（3 条）
1. **策略身份 = 规范化哈希**：PolicyHash 用 allow-list 投影 + 穷举 destructure + 凭据排除，实现"可审计且不泄露"的运行指纹
2. **配置不能自我授权**：实验性后端 opt-in 是调用方参数（JSON 解析器硬编码 false），遥测不是策略字段
3. **时间线逃逸防御**：Bubblewrap `--die-with-parent` 防超时 kill 后后台后代继续运行在已拆除的网络强制之外
