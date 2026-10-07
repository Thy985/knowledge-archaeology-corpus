# PullFrog 考古索引

**一句话定位**：PullFrog = 权限分层的 agent 编排 meta-harness —— 把托管编码 agent（Claude Code / Codex / OpenCode）挂到受控 MCP 工具面 + 沙箱 shell + 生命周期门控上，在不可信仓库上安全地执行可信 agent。

**考古日期**：2026-10-02 · **commit**：`d287f81e93e245845aa25b0916417a4216f37dad` · **skill version**：3.2 · **run**：ARCH-2026-10-02-001

## 为什么值得读

在"可信 agent + 不可信 repo"的安全模型上，PullFrog 给出了一个完整可运行的答案：四类 token 分层（可渗漏/不可渗漏分化）、shell 三态权限 + mount/PID 命名空间沙箱、subagent 状态变更门控、ASKPASS 凭证生命周期、双超时失联治理、post-run 可见交付门控，以及把安全属性变成 CI 断言的三层对抗测试。代码注释里锚定了 ~30 个真实事故（#760/#844/#862/#891/#906/#938/#964/#1077/#1085/#1093/#1115/#1120/#1121/#1139/#1146/#1171/#1179 等）——是失败驱动设计的富矿。

## 产物清单

| 目录/文件 | 内容 |
|-----------|------|
| `00_overview.md` | 定位、架构三层视图、核心发现 5 条、关键争议、已知代价 |
| `01_project-layer/` | 项目地图（模块行数/核心数据结构/生命周期/测试三层/配置/权限机制） |
| `02_engineering-knowledge/` | EK Graph 43 条（六类边 links，全部带证据） |
| `03_knowledge-layer/` | Generalized KO 8 个（R1-R4 聚合规则，L3/L4/L5） |
| `04_flow-atlas/` | 七类流（Control/State/Data/Evidence/Authority/Memory/Policy） |
| `05_candidates/` | 8 个未验证假说 + 2 个暂定模式 |
| `06_validation/` | 六类 Auditor + Blind Reconstruction + 判定统计 + Reconciliation |
| `archaeology-runs/ARCH-2026-10-02-001/` | run_metadata.yaml + snapshot_artifact.md |

## 三层配比

- Engineering Knowledge：43 条（全带 links，游离 0）
- Generalized KO：8 个（R1×2 / R2×2 / R3×3 / R4×2，簇均规模 4.6）
- Candidates：8 假说 + 2 暂定模式

## 关键争议（需人审）

1. stop hook 已禁用（#714 审计 8/9 foot-gun）——用户配置的 stop script 不再把关（C-02）。
2. Router 分层架构（jev|scorer|heuristic|fixed）在服务端，无法从本仓库验证（C-06）。
3. KO-01 Fail-Closed 已按反例（EK-20 静默回退）收紧为"安全不变量失效必须失败"。
