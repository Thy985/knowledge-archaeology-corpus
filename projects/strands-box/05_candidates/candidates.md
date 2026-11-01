# 05 Candidates — strands-box（ARCH-2026-10-23-001）

> 不能确认的内容留在候选。Hypothesis ≠ Fact；Cross-project Candidate 不写成 Principle。

## C-B01 · Linux 命名空间后端的生产级成熟度（Hypothesis）
- **现状**: backend/linux/namespace/* 实现明确（authority/netns/probe/reaper/syscall/view + contain_trampoline）；README 说"currently supports local execution on macOS with Apple silicon"；containment tests 有 gateway_fail_closed_linux / inherited_handles_linux / contains_exec_target_linux
- **缺失证据**: Linux 端 workload 黑盒套件覆盖度（test-workload 是否在 Linux CI 全量跑）；namespace 后端 vs seatbelt 的成熟度对比
- **验证路径**: .github/workflows/ 中 Linux runner 矩阵；test-workload 的 CI 平台条件
- **判定**: 双后端实现为 Fact；"Linux 生产级"为 Hypothesis（README 显式 preview macOS）

## C-B02 · dogwood-local-engine 跨进程历史一致性（Hypothesis）
- **现状**: durable assembly 追加事件 + 私有 history 文件恢复（Cargo.toml 注释）；reload 与在飞决策串行化
- **缺失证据**: 多 box/多进程共享同一策略时的历史隔离/一致性语义；历史文件增长边界（无限事件？）
- **验证路径**: dogwood-local-engine crate 文档（外部依赖源码）；telemetry 与 history 的关系
- **判定**: Hypothesis（引擎为外部 crate，本次考古未深入其源码）

## C-B03 · 性能断言未基准（Observation）
- **现状**: tenets 性能表：policy <1ms（"asserted, not yet benchmarked"——仓库自己标注未测）、startup <50ms、mem <20MB
- **缺失证据**: benchmark 数据（crates/box-bench 存在但被 exclude）
- **验证路径**: crates/box-bench 基准跑（本环境 Linux 可部分跑）
- **判定**: 目标声明为 Fact（tenets 原文）；达成度为未验证

## C-B04 · 生态采纳（Hypothesis）
- **现状**: AWS 官方博客（2026-10-07）+ strandsagents.com + Discord；harness agnostic 声明（Strands Harness/LangChain/Claude Agent SDK）
- **缺失证据**: 第三方 harness 集成证据（examples/ 为仓库自带）；dev preview 阶段
- **验证路径**: examples/ 目录第三方适配；GitHub issues 生态反馈
- **判定**: "生态采纳"未验证（与 aegis AGEF/AGP 同理：自我声明 ≠ 生态事实）

## C-B05 · 三种治理位置的跨项目对照（Cross-project Candidate）
- **现状**: Box（OS 遏制 + 进程外 PDP）/ MXC（平台级多后端三模式）/ agent-aegis（进程内插桩 approval gate）/ deepseek-harness（sandbox+approval seam）
- **假说**: 治理位置谱系 = OS 遏制（Box/MXC）→ 进程边界（dsh）→ 进程内插桩（aegis）；**低摩擦 vs 强制性的权衡**随位置移动；aegis 低摩擦但依赖 agent 进程内被 patch，Box 强制但需 operator 显式声明全部访问
- **验证路径**: 并置四 corpus 条目（strands-box/dogwood/MXC 对照卡/agent-aegis/deepseek-harness）做三方治理位置分析
- **判定**: Cross-project pending（单项目证据不足升 Principle）

## C-B06 · MCP broker 的信任模型细节（Hypothesis）
- **现状**: docs/design/mcp.md（stdio 自沙箱/http 外部检查）；双门决策 ADR；"every MCP server is one configuration"（名字来自 box.toml 非请求值）
- **缺失证据**: broker socket 的进程边界实现细节（本地 socket 连接方身份）；mcp-discovery 机制（docs/design/mcp-discovery.md）与 policy 生成的交互
- **验证路径**: mcp broker 实现（crates/box 或独立模块）深读
- **判定**: 文档模型为 Fact；socket 身份实现细节未深读

## C-B07 · reload 与在飞决策的一致性窗口（Hypothesis）
- **现状**: reload 在 authority 内建锁上串行化 in-flight 决策（policy.rs:228-245）；"The caller owns the question of who may ask"
- **缺失证据**: 长决策（如大文件 fs:read）中途 reload 的语义；reload 失败时的回滚状态（PolicyStagingError 语义）
- **验证路径**: authority.rs（1581 行）ProposalState 实现深读
- **判定**: 串行化机制为 Fact；长决策一致性窗口为 Hypothesis
