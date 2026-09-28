# Microsoft Agent Framework — Knowledge Archaeology 总览（ARCH-2026-09-29-001）

> run_id: ARCH-2026-09-29-001 ｜ commit: 95e711a6280d0ed7bafa5bafb8ff38eba4cffd3e ｜ skill: knowledge-archaeology v3.2
> repository: https://github.com/microsoft/agent-framework.git ｜ mode: initial

## 一句话定位
Microsoft Agent Framework：构建、编排、部署 AI agents 与多代理系统的生产级框架（MIT，Python/C# 双栈，2026-09-28 仍活跃）。本文考古聚焦 **Agent Harness 抽象 + 多代理编排 + 原生记忆 + FIDES 确定性安全**（Python 核心面，C# 做同构对照）。

## 核心认知（L4 摘要，详见 03 层）
1. **Trusted Ladder（信任阶梯）**：框架把"可验证确定性"与"模型自由性"分层——工具审批（确定性规则/预算）、文件路径（规范化防穿越）、标签传播（信息流控制）都是**确定性机制**；LLM 的创造力被约束在"经过审批/隔离/审计"的执行面上。**智能 ≠ 权威**（与 AProver/Aigis/Tafcm 三项目收敛）。
2. **确定性审批循环**：ToolApprovalMiddleware 不是"问一次"，而是**循环内状态机**（session 持久化审批状态 + 函数调用预算 + 自动批准队列 + 二次审批），直到所有工具调用被批准或转交人类。审批语义有 authoritative 边界（session 归属决定审批是否权威）。
3. **FIDES 标签即权威**：IntegrityLabel/ConfidentialityLabel 三级传播（内嵌 > source_integrity 声明 > 输入标签/默认），untrusted 内容**物理隔离**（变量间接）+ 可被**隔离执行**（quarantine）——提示注入防御是确定性信息流控制，不是 prompt 工程。
4. **记忆 = 文件/仓库 + 可信注入**：file memory（topic markdown）与 Cosmos 原生记忆并行；检索到的记忆**以 user 角色 untrusted 注入**（防存储提示注入）；user_id 决定跨会话记忆作用域（稳定 user_id 才有跨会话记忆）。
5. **Harness 循环 = 受控的坚持**：max_iterations 安全上限 + should_continue predicate + fresh_context（session 快照重置）→ 循环的"每次迭代上下文干净性"由框架保证；pending approval 是循环的**逃生舱**（转交人类）。
6. **CodeAct：模型写代码 ≠ 模型掌权**：模型生成代码在沙箱（Hyperlight）内执行，代码相对宿主 untrusted，框架只负责审批/能力/遥测——**沙箱边界是信任模型的硬约束**（"If a backend cannot provide isolation appropriate for its trust model, it is not a suitable CodeAct backend."）。

## 三层配比（聚焦面）
| 层 | 数量 | 说明 |
|---|---|---|
| Facts（01 层内嵌） | 32 | 可溯源到文件/行 |
| Engineering Knowledge | 46 | EK Graph 六类边，links 全覆盖 |
| Generalized KO | 8 | R1-R4 聚合规则 |
| Candidates | 5 | 未验证假说 |

## 关键事实锚点
- HEAD commit 95e711a6（2026-09-28 18:43 +0100，"Update version for 1.23.0 release (#8806)"）
- core 包 version 1.19.0（pyproject）；136MB / 5412 文件 / 96+8 测试文件
- Python `_harness/` 与 C# `Harness/` 八模块一一对应（AgentMode/BackgroundAgents/FileAccess/FileMemory/FileStore/Loop/Todo/ToolApproval）
- 本机实测 pytest：file_access 路径安全 7 passed、tool_approval 审批语义 7 passed（S5 运行时证据）
- ADR 0024（FIDES 确定性提示注入防御，status=proposed，2026-01-14）与 security.py 实现一致
