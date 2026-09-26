# 05 · Candidates（未验证假说 / 跨项目假设）— browser-use

> 不能确认的内容一律留在候选；Hypothesis 不冒充 Fact；Cross-project Candidate 不写成已验证 Principle。

## C-01 软门控/硬门控分工是跨项目 agent 治理模式（Cross-project Hypothesis）
- **状态**: Hypothesis（Cross-project validation pending）
- **本仓库证据**: EK-03（循环检测"从不阻断"+ nudge 注入）+ EK-04（预算/失败硬门控）
- **跨项目对照**: agent-governance-toolkit（确定性策略引擎 0.1ms p99——硬门控路线）；OpenClaw（sandbox 默认 deny——硬门控）
- **缺失证据**: 未在第三个项目验证"软门控提升完成率"的因果；未量化软 vs 硬对任务成功率的影响
- **验证路径**: 对照考古 agent-governance-toolkit 的 policy 引擎 vs browser-use 的 nudge 机制，比较失败路径

## C-02 OSS 漏斗产品化模式（第二实例，Cross-project Pattern 候选）
- **状态**: Pattern（2 实例，S8 级证据不足，pending 第三实例）
- **本仓库证据**: EK-16（AGENTS.md 推荐自有模型 + use_cloud + API key）
- **跨项目对照**: mem0（已考古，同构：OSS SDK→平台→自有模型）——两实例同构
- **缺失证据**: 第三实例（OpenClaw 的商业模式？）；收入/转化数据不可见（私有）
- **验证路径**: 考古 OpenClaw 或 hermes 的 README/AGENTS 商业化痕迹；或等待第三个开源 agent 项目入库

## C-03 DOM 观察端压缩是"环境传感器超载"的通用解法（Cross-project Hypothesis）
- **状态**: Hypothesis
- **本仓库证据**: EK-06（DOM → 交互元素 + selector_map，缓存复用）
- **跨项目对照**: Webwright（browser agent，同域但未考古）；IDE agent（代码树压缩——未入库）
- **缺失证据**: 仅 browser-use 一个强实例；Webwright 压缩策略未核验
- **验证路径**: 考古 microsoft/Webwright（候选卡 B 级）对比 DOM 压缩策略

## C-04 Beta 终端 Agent 是 browser-use 的第二能力面（Scope-uncertain）
- **状态**: Observation（beta 未完全验证）
- **本仓库证据**: EK-15（RustSdkClient + agent tools + ripgrep 检测）
- **不确定**: beta/service.py 6,810 行是最大文件，但测试覆盖有限；Rust SDK 二进制不在本仓库（外部终端工具）
- **缺失证据**: 终端 agent 与浏览器 agent 的编排方式未深读（时间盒限制）；beta 质量门未验证
- **验证路径**: 后续考古轮补 beta 深读，或运行 beta 模式实测

## C-05 telemetry 驱动产品化的数据闭环（Tentative Pattern）
- **状态**: Hypothesis
- **本仓库证据**: EK-17（posthog + device_id + ANONYMIZED_TELEMETRY 默认 true）
- **跨项目对照**: OpenClaw（遥测默认？未核验）；mem0（平台数据侧——已考古但未聚焦）
- **缺失证据**: 遥测数据如何回流产品决策不可见；"匿名默认开"是普遍 OSS 实践还是 browser-use 特有
- **验证路径**: 对照考古 agent 类项目的 telemetry 默认策略
