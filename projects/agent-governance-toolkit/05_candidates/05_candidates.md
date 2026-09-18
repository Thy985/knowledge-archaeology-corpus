# 05 — Candidates（未验证假设 / 跨项目候选）

> Hypothesis 标记，不冒充知识。每条含：候选内容 / 当前证据 / 缺失证据 / 验证路径。

## C-01 [Hypothesis] 确定性治理（AGT）与启发式治理（AI Protector）存在最优折中点
- **内容**：AGT 把模型完全移出决策路径（纯确定性）；AI Protector 在路径内放本地 ML 权衡（HARM_ML_MODE）。可能存在最优混合：确定性 fail-closed 基座 + 有限启发式增强（可解释）。
- **当前证据**：AGT EK-33（无 LLM 决策路径）+ AI Protector EK-27（ML 权衡）
- **缺失证据**：无第三项目在"同一威胁集"下对比两种架构的漏报/误报
- **验证路径**：用 benchmarks/prompt-injection 280 smoke corpus 对两个项目跑同一基线

## C-02 [Cross-project Candidate] "治理栈从 OS 借力"是 Agent 平台品类共性
- **内容**：AGT（执行环/SRE/供应链）与 Guardian/AI Protector 都部分采用系统软件词汇（审计链、sidecar、proxy gate）。但完整度差异大——可能 AGT 是"全栈参考实现"而其余是"点治理"。
- **当前证据**：KO-06（AGT）+ 前三项目考古
- **缺失证据**：同类项目样本只有 4 个（本品类四选实测）
- **验证路径**：更多 agent-governance 品类项目入 corpus 后做跨项目聚合

## C-03 [Hypothesis] fail-closed 的"可见性优势"在 agent 场景被放大
- **内容**：传统安全 fail-open 静默；agent 场景因"agent 可自我修复/自我调整"，静默绕过会更快被利用（agent 会反复尝试）。fail-closed 的可见误拒给运维即时反馈。
- **当前证据**：ADR-0013 论证（"fail-open errors are silent and may go undetected"）
- **缺失证据**：无运行时数据证明 agent 场景的利用速度差异
- **验证路径**：redteam benchmark（tests/redteam/）中对比 fail-open/fail-closed 的绕过成功率

## C-04 [Hypothesis] trust ceiling 单调收敛可能过度限制合法"能力委托"
- **内容**：子代理 ≤ 父代理信任是防提权安全网，但"强工具 + 弱信任父"场景（如用户委托给低信任编排 agent 执行高权限任务）可能被误伤——委托链存在合法的信任"跳跃"场景。
- **当前证据**：ADR-0016 决策 + 权衡说明（"child agents in trusted environments may be artificially constrained"）
- **缺失证据**：无生产部署数据评估误伤率
- **验证路径**：AGT Studio / 生产用户的 delegation chain 遥测（若公开）

## C-05 [Cross-project Candidate] 审计"可证明性三层"（Merkle→BOM→互操作记录）是合规 agent 系统基线
- **内容**：Guardian 有审计链（单层），AGT 有完整三层。可能"三层是监管要求下的必然收敛"。
- **当前证据**：KO-04 + Guardian EK-06
- **缺失证据**：只有 2 项目对照
- **验证路径**：corpus 内更多合规敏感 agent 项目对照

## C-06 [Hypothesis] ACS 的 intervention points（agent_startup/input/pre_model_call/pre_tool_call/tool_result）可能是策略作用点的"标准命名"
- **内容**：AGT 用命名干预点切 agent 生命周期；AI Protector 用 pre_llm/pre_tool/post_tool 等 stage。可能形成事实标准。
- **当前证据**：EK-14 + AI Protector stage 体系
- **缺失证据**：业界未统一，样本少
- **验证路径**：跨项目 stage 词汇表聚合（corpus 横向检索）

## C-07 [Hypothesis] Rego/DSL "静默不匹配"是策略即代码的普遍陷阱
- **内容**：AGT 的 DSL 限制 + Rego 输入包装导致规则"永不匹配但不报错"。其他策略系统（K8s admission、OPA 用户）可能同样踩坑——"策略测试必须验证规则真命中"是普适纪律。
- **当前证据**：EK-22b（AGT 详细警告）
- **缺失证据**：非 AGT 场景实例
- **验证路径**：corpus 内其他含策略 DSL 项目对照

## C-08 [Scope-uncertain] 全仓 13,834 测试函数 vs README 992 conformance 的缺口
- **内容**：README 声称 992 conformance tests（10 specs）；代码扫描出 13,834 个 test 函数（跨所有语言包）。差异可能来自：conformance 特指 spec 驱动的子集，其余是各包单元/集成测试。但精确归属未核实。
- **当前证据**：README S7 声称 + 代码扫描 S3
- **缺失证据**：conformance 测试与总测试的精确对应关系
- **验证路径**：跑 policy-engine/spec conformance 测试套件核对

## C-09 [Hypothesis] policy-engine/core 的 deprecation shim 造成"仓库源码 ≠ 运行时逻辑"审计盲区
- **内容**：决策逻辑实际在 crates.io agent_control_spec（依赖 agent-hooks 契约），本地 core 只 re-export。考古/审计必须跟踪到外部 crate 版本——"仓库快照不能证明决策逻辑"。
- **当前证据**：EK-16（lib.rs deprecation 注释）
- **缺失证据**：外部 crate 与本地 spec 的一致性声明
- **验证路径**：核对 Cargo.toml 锁定版本 + crates.io 源码

## C-10 [Cross-project Candidate] "包整合期 CI 退化"（PR #2794）是 monorepo 演进的常见事故
- **内容**：删 dev extra → .[dev] no-op → 测试收集全挂。合并多包时，依赖清单/测试可运行性是最易被静默破坏的资产。
- **当前证据**：EK-28（pyproject 注释自述）
- **缺失证据**：其他 monorepo 同类事故样本
- **验证路径**：corpus 内 monorepo 项目（deepseek-harness 等）对照
