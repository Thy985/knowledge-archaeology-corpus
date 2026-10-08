# ThinkingBox 知识考古索引

> 一句话定位：**有状态业务工作流的 agent 沙箱与评测框架**——MCP server 即被测世界，TestScript 断言直接读写状态真相评分，pass^k/goldilocks 统计层对抗 pass@1 幻觉（arXiv:2608.19741 "One Success Isn't Reliability"，MS Copilot Studio RL 团队）。

## 考古信息

- 仓库：https://github.com/microsoft/thinkingbox.git（commit `892964e`，v0.1.0，2026-10-03，MIT）
- 考古日期：2026-10-08 · run_id：`ARCH-2026-10-08-001` · skill：knowledge-archaeology v3.2
- 三仓库边界：thinkingbox（本文框架）/ thinkingbox-data（语料+servers+support）/ thinkingbox-training（RL 后训练）

## 产物清单

| 目录/文件 | 内容 |
|---|---|
| `metadata.yaml` | 考古元数据（commit/run/summary/判定统计） |
| `00_overview.md` | 总览：定位、三仓库边界、外部宣称边界、关键发现、corpus 连接 |
| `01-project-layer/project_layer.md` | 项目地图：架构/模块/生命周期/配置/依赖/入口/治理 |
| `02-engineering-knowledge/ek_graph.md` | 56 条 EK（EK Graph 六类边：mechanism/subsystem/causal/dependency/constraint/contrast） |
| `03-knowledge-layer/knowledge_layer.md` | 9 个 KO（R1×2/R2×2/R3×2/R4×3 聚合规则矩阵） |
| `04-flow-atlas/flow_atlas.md` | 七类流（Control/State/Data/Evidence/Authority/Memory/Policy）+ Flow→KO 交叉校验 |
| `05-candidates/candidates.md` | 15 候选（外部宣称隔离区 C-01..06 / 跨项目假说 C-07..10 / 暂定 C-11..13 / 已排除 C-14..15） |
| `06-validation/validation_report.md` | 六审计 + 质量指标 + Reconciliation（含独立 Auditor 修正合并） |
| `06-validation/independent_validation_report.md` | 独立 Auditor 盲重建报告（F1-F10，10 CONFIRMED/3 PARTIALLY/4 MISSING） |
| `archaeology-runs/ARCH-2026-10-08-001/` | 本轮原始产物（job_manifest / snapshot_artifact / run_metadata / independent_validation_report） |

## 关键发现（Top 6）

1. **状态真相评分**：agent 行为按 effects（MCP server 状态）与 proxy 调用日志双源取证，不按叙述评分
2. **停止闸门**：八值 finish_reason + agent turn 上限优先 + watchdog 超时 terminate
3. **pass^k 统计核心**：(c/n)^k 有意有偏估计 + goldilocks P(zone|data)≥0.95——单次成功≠可靠性
4. **副作用安全**：工具超时绝不重试（max_retries_timeout=0，防重复副作用）；LLM 读取可重试（分层重试对照）
5. **工具面声明化**：schema 声明≠授权，visible_tools 过滤 + 同名裁决 + fixture 后门
6. **注入防护三件套**：内容 sanitize（6 组转义）/ judge 输入不可信声明 / 输出 safe_tag_encode

## 考古日期/版本

- 首次考古：2026-10-08（initial，commit `892964e7226044e5463ad188da119df838f4bec1`）
- 后续刷新：未安排（候选中 C-01..06 依赖 thinkingbox-data，若语料仓库开放可做 refresh）
