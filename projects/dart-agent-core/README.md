# dart-agent-core 项目考古索引

**一句话定位**：Dart/Flutter 生态 mobile-first/local-first agent 运行时 + agent 评测子系统一体化库（StatefulAgent + eval.dart）。

- **考古日期**: 2026-10-10
- **版本**: 2.1.7（commit `43c11f1`）
- **Skill Version**: knowledge-archaeology v3.2
- **run_id**: ARCH-2026-10-10-001

## 产物清单

| 层 | 文件 | 内容 |
|----|------|------|
| 00 | `00_overview.md` | 一句话定位 + 三条核心认识 + 产物清单 |
| 01 | `01-project-layer/project_layer.md` | 两入口/模块地图/双生命周期/状态字段/配置/权限治理/依赖 |
| 02 | `02-engineering-knowledge/ek_graph.md` | 60 EK（EK Graph 六类边，links 覆盖率 100%） |
| 03 | `03-knowledge-layer/knowledge_layer.md` | 8 KO（R1×3/R2×2/R3×2/R4×2 + 五条件自检） |
| 04 | `04-flow-atlas/flow_atlas.md` | 七类流（Control/State/Data/Evidence/Authority/Memory/Policy） |
| 05 | `05-candidates/candidates.md` | 9 Candidates（C-01~C-09，全部 Hypothesis/未决） |
| 06 | `06-validation/validation_report.md` | 六 Auditor + 盲重建 + 反例预算制 + 质量指标 |
| 06 | `06-validation/independent_validation_report.md` | 独立 Auditor 盲重建（9 CONFIRMED/1 PARTIAL/1 DOWNGRADED/1 REVIEW） |
| runs | `archaeology-runs/ARCH-2026-10-10-001/` | job_manifest / snapshot_artifact / run_metadata / independent_validation_report |

## 关键认知（速览）

1. **评测与运行时共享"非确定性保护"心智**（KO-07）：trialSalt 独立缓存槽 + record/replay 严格重放 + systemPrompt/tools 版本历史——"要测量/重放必须先版本化记录"。
2. **失败不伪装成功是全库最一致不变量**（KO-05）：跨 agent 运行时/评测/子代理/judge 四子系统。
3. **工具面=控制面解耦**（KO-01）：hook 是唯一行为控制通道；controller 只观察。
4. **pass^k empirical 动机**（C-01）：与 thinkingbox 同公式、动机分化（CI 报告直观 vs 防 pass@1 幻觉）。

## Corpus 连接

- thinkingbox（评测统计同构）/ agentevals / deepseek-harness（评测框架线）
- mcp（桥工具对照）/ opencode / hermes-agent（agent 运行时线）
- **Tafcm（用户 Dart 项目）直接复用对象**
