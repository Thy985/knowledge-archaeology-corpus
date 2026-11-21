# Validation & Evidence — Strands Box（ARCH-2026-10-20-001）

## Truth Audit（真值）
- 全部 EK/KO/Flow 事实可回溯 repo 实盘（README/Cargo.toml/AGENTS.md/docs/design/*/crates/*/test-*），HEAD `069c39c`
- 文档主张 vs 代码实现分离引用：设计锚（decisions.md）标记为设计意图，源码（crates/）标记为实现事实
- 0 CONTRADICTED（本轮主考古视角）；独立 Auditor 判定见 independent_validation_report.md

## Coverage Audit（覆盖）
- 强制子系统覆盖：policy crate（cap.rs/adapters/address）、egress-gateway（capability/decider/credential）、containment（macOS Seatbelt + Linux namespace 后端）、credentials（attach/legs）、telemetry（collector/receive/export）、box CLI（run.rs/record/config）、shell crate、MCP 设计（mcp.md）、测试体系（test-integ/test-workload）
- 未深读（声明）：docs/user/tutorials/*（教学路径）、.agents/pocs（POC 排除）、test-integ 各用例内部（黑盒运行未复跑）

## Flow Audit（流真实）
- 七类流 Edge 全部从 architecture.md/decisions.md/run.rs/collector.rs/receive.rs/decider.rs 导出
- 关键 Edge 复核：run 定序 7 步（run.rs:80-103 + architecture.md）、collector drained 唯一出口（collector.rs:345）、CapabilityOutcome fold（decider.rs:10-35）、receive strip/stamp（receive.rs:42-71）

## Abstraction Audit（升维）
- L4 仅 KO-B01/KO-B02/KO-B04（均有跨子系统一致性证据）
- KO-B03/KO-B05 停在 L3 Pattern（跨项目验证 pending 标注）
- 无"项目经验→通用 Principle"单案例升维

## Counterexample / 反例
- KO-B01 反例：Box 自身 interpreter 无 credential floor（暴露面诚实保留，floor 不保护凭据）
- KO-B02 反例：inject=always 无占位符 advisory override（attach.rs:381）——注入点仍可能被误配置，但不算授权
- KO-B03 反例：direct fs grant 不进历史（agent 自己 syscall 读文件不产生决策事件）——历史只含经 enforcement point 的操作
- 0 反例搜索证据：对每条 L3+ KO 至少一次定向反例攻击（见各 KO 反例压力节）

## Epistemic Audit（认知状态）
- Fact（L1）与 Observation（L2）标注严格；Hypothesis 全部进 Candidates（C-B01~B07）
- Cross-project Candidate 未写成已验证 Principle（C-B03/B04/B05/B06 均标注 pending）

## 质量指标
- EK 20（B01~B20，EK Graph）；KO 5（R1×2/R2×1/R3×2）；Candidates 7；Flow 7 类
- links 覆盖率 100%；游离 0%；聚合覆盖率 100%
- 证据引用 ~70 处（文件:行/文档锚）

## Blind Reconstruction 预留
- 独立 Auditor 不引用本文，独立重读仓库（见 independent_validation_report.md）；修正经 reconciliation.md 落地
