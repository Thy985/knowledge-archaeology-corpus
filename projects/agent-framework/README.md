# agent-framework — Archaeology Corpus Entry

一句话定位：Microsoft Agent Framework——构建/编排/部署 AI agents 与多代理系统的生产级框架（MIT，Python/C# 双栈），考古聚焦 **Agent Harness 抽象 + 多代理编排 + 原生记忆 + FIDES 确定性安全**。

- 考古日期：2026-09-29（run ARCH-2026-09-29-001，initial）
- repository：https://github.com/microsoft/agent-framework.git
- commit：95e711a6280d0ed7bafa5bafb8ff38eba4cffd3e（core 1.19.0 / dotnet 1.23.0）

## 产物清单
| 层 | 文件 | 内容 |
|---|---|---|
| 00 | `00_overview.md` | 总览（核心认知 + 三层配比 + 锚点） |
| 01 | `01_project-layer/README.md` | 项目地图（定位/结构/模块/生命周期/测试/配置/治理/依赖/ADR） |
| 02 | `02_engineering-knowledge/README.md` | EK Graph（46 条 EK，六类边） |
| 03 | `03_knowledge-layer/README.md` | Generalized KO（8 个，R1-R4 聚合规则） |
| 04 | `04_flow-atlas/README.md` | 七类流（含 Policy Flow + Flow→KO 交叉校验） |
| 05 | `05_candidates/README.md` | 5 个未验证假说 |
| 06 | `06_validation/README.md` + `independent-audit.md` | Validation 报告 + 独立盲重建（12 CONFIRMED / 0 CONTRADICTED） |
| runs | `archaeology-runs/ARCH-2026-09-29-001/` | job_manifest / snapshot_artifact / run_metadata / package 原始产物 |

## 核心认知（五条）
1. **确定性审批循环**：审批 = 会话内状态机（预算/队列/二次审批/逃生舱），非一次性 yes/no。
2. **FIDES 标签即权威**：untrusted 内容物理隔离 + quarantine 隔离执行——提示注入防御是信息流控制，不是 prompt 工程。
3. **记忆注入是安全决策**：user_summary 以 untrusted user 角色注入，防存储提示注入。
4. **循环受控坚持**：max_iterations 先短路 + fresh_context 上下文重置 + 结构化停止谓词。
5. **CodeAct 沙箱边界**：模型代码 = untrusted，后端隔离能力是硬约束（ADR 0038）。

## 关键争议
- ADR 0024 为 proposed 状态但 security.py 已实现（设计意图领先于 ADR 定稿）。
- quarantine_client 为全局单例，多租户隔离风险（C-03）。
