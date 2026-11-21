# Candidates — MXC 未确认内容（Hypothesis / Cross-project pending）

> 与 KO 严格区分：以下全部是**不能确认**的内容，不写成已验证知识。

## C-M01 · Learning Mode 最小权限自助闭环（Hypothesis）
- **主张**: plm denials.json + compatibility adjusted-config generator 完成后，用户可"记录 → 自动生成 → 审阅 → 应用"最小权限策略，无需手工枚举路径
- **当前证据**: plm readme（denials.json + "compatibility adjusted-config generator" + "Review generated policy before using it"）；logging-access-denied.md（"the machinery that collects and surfaces the resulting denial events is layered on top in later work"——**明确标注后续工作**）
- **缺失证据**: 生成→应用闭环的端到端实现/测试；非 Windows 平台支持
- **验证路径**: 检查 plm 生成器与 policy 应用链路 + sdk-v1 金标准测试

## C-M02 · MicroVM/Hyperlight 是隔离谱系终态（Hypothesis）
- **主张**: MicroVM（Nanvix）/Hyperlight 提供"最强隔离"，未来默认后端的演进方向
- **当前证据**: backend_registry.rs NotDetected("no hypervisor-presence detector yet; see the follow-up items in docs/development/plans/backend-support-probe-api.md")——experimental + 无 probe
- **缺失证据**: 性能/兼容性/生产就绪度
- **验证路径**: backend-support-probe-api 计划落地 + 运行时验证

## C-M03 · MXC 生态集成用法的策略生成闭环（Hypothesis）
- **主张**: Copilot/OpenClaw/Codex 等接入方会用 Learning Mode 生成各自最小权限策略
- **当前证据**: 官方博客生态矩阵（仓库外）；本仓库无集成示例
- **缺失证据**: 仓库内证据（samples/ 未见集成代码）
- **验证路径**: 观察生态接入 commit（openclaw 已考古 corpus，可对照）

## C-M04 · FS deny/grant 交互 backend-specific 的语义一致性（Hypothesis）
- **主张**: 各后端对 deniedPaths vs readwritePaths 冲突的裁决不一致，存在跨后端行为差异
- **当前证据**: policy.md（"Enforcement and interactions with grants are backend-specific"——**官方承认 backend-specific**）
- **缺失证据**: 各后端冲突裁决的具体差异清单；无跨后端一致性测试
- **验证路径**: 逐后端策略冲突矩阵测试

## C-M05 · 进程内 SDK 嵌入 vs 独立 executor 的威胁模型差异（Hypothesis）
- **主张**: SDK 嵌入应用进程（in-process）与 wxc-exec 独立进程的隔离强度/攻击面不同，威胁模型应分开声明
- **当前证据**: README（"SDK dependency that builds into your app" + "Non-SDK consumption: executor binaries"）；无威胁模型文档区分
- **缺失证据**: 威胁模型文档；攻击面对比
- **验证路径**: 查 docs 威胁模型/审计文档

## C-M06 · WSLc portMappings 仅 TCP 的演进方向（Hypothesis）
- **主张**: UDP 映射是后续需求，1.0 契约暂不支持
- **当前证据**: policy.md（"portMappings（array of windowsPort / containerPort…protocol: TCP only）" + "Provision port mappings are not available in 1.0"）
- **缺失证据**: roadmap 中对 UDP 的承诺
- **验证路径**: dev 契约 1.1.0-alpha schema 检查

## C-M07 · 跨项目假说：契约三轴 vs 统一 Authority（Cross-project Hypothesis）
- **主张**: MXC"契约/运行时/授权三轴分离"与 strands-box"三 PEP 共享唯一 PDP（Three views of one authority）"是**同一治理模式的两端**：前者在 SDK 入口面分离决策，后者在执行点统一决策
- **当前证据**: MXC versioning.md 权威表；strands-box box/AGENTS.md:292（10-23 run）
- **缺失证据**: 两项目间无直接证据桥——分离 vs 统一何者更优未验证
- **验证路径**: 需要第三项目（如 deepseek-harness 权限预设）对照；**未验证前不得写成 Principle**

## Candidates 统计
- 7 条（C-M01~C-M07）；全部标 Hypothesis / Cross-project pending；无一条冒充已验证知识
