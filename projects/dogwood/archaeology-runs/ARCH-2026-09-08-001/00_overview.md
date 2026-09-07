# 00 · Overview — Dogwood（ARCH-2026-09-08-001）

> 核心问题：**"Dogwood 让我们认识到了什么？"** —— 一个给 AI Agent 加"时间维度治理"的运行时验证语言，如何把"动作必然合规"从启发式变成可判定检查。

## 一句话定位

Dogwood 是 AWS 开源的 **agent 运行时验证治理语言**（Apache 2.0，v1.0.0，2026-08-06 开源）：Cedar 派生语法 + **时间条件**（`since`/`formerly`/`once`/聚合），在每次工具调用前对 agent 的**近期事件历史**做判定；策略可 **compile-to-Cedar**（temporal/provider 字段变 `context.*` 槽），policy/temporal 引擎可插拔。

## 为什么值得考古（对个人知识系统的价值）

1. **"三段式可证明安全"的运行时侧**：skillfortify（声明侧静态）↔ Dogwood（运行时侧验证）——候选卡 [cand]agent-formal-verification 判定的工程落点。
2. **策略即代码 + 调用前判定**：EP-002 "Permission Is Security Boundary" 的工业级参考实现（permit/forbid + when/unless + 时间窗口）。
3. **时间状态治理的完整设计**：事件 schema（decision vs history points）、max_window 回看上限、pin 分区、InMemoryTemporalEngine——"agent 过去行为如何进入决策"的教科书。
4. **诚实的生产边界声明**：README 尾部 6 条"参考解释器 NOT production"限制（Rhai 沙箱无默认限制、审计日志缺失、多租户无隔离、错误泄露）——参考实现的自我审计，与 omnigent OBSERVABILITY 同族。

## 认知核心（详见 03）

- 时间条件是治理的关键维度：一次性判定（permit/deny）不足以约束 agent——需要"过去发生了什么"（since/formerly/once/窗口聚合）
- 编译到既有策略语言（Cedar）而非新造运行时：降低采纳成本、复用生态
- 参考实现的诚实边界：语义保证（language contract）与实现选择（deny-on-error 是参考实现行为，不是语言契约）严格分离
- 注入面即攻击面：provider 脚本（Rhai）与事件字段都被视为不可信输入

## 关键数字（全部可追溯到仓库）

| 项 | 值 | 来源 |
|----|-----|------|
| 规模 | 84 rs 文件 / 46,208 行 | find + wc |
| 版本 | v1.0.0（workspace.package） | Cargo.toml |
| HEAD | c6237c88099b3f492ecc5fcee42df06a19224b97 | git rev-parse |
| workspace | 3 crates（dogwood-language / dogwood-cli / dogwood-docs） | Cargo.toml |
| 语言 | Rust 2024 edition | Cargo.toml |
| 开源时间 | 2026-08-06（AWS 博客） | README + 候选卡 |
| 文档 | mdBook 12 章 guide（含 formal spec 08） | dogwood-docs/guide/ |
| 测试 | 30+ 测试文件（boundary/cedarify_adversarial/fail_closed/incremental_lowering/expected_failures/docs_as_tests） | dogwood-language/tests/ |

## 产物清单

| 文件 | 内容 |
|------|------|
| 01_project-layer.md | 项目地图 |
| 02_engineering-knowledge.md | EK Graph（28 EK + 六类边） |
| 03_knowledge-layer.md | Generalized Knowledge（6 KO + 4 CM + 3 M） |
| 04_flow-atlas.md | 七类流 |
| 05_candidates.md | 未验证假说与跨项目候选 |
| 06_validation.md | 验证 + 质量指标 |
| run_metadata.yaml | run 元数据 |

## 免责声明

- 快照基于浅克隆 HEAD c6237c88。
- **测试未运行**：环境无 Rust 工具链（cargo 不存在），46k 行无法编译验证；"测试揭示行为"类证据为**测试意图阅读**（S3 级），非运行验证（S4 级）——已在 06 如实标注，不弱化标准。
- 参考解释器明确 **NOT intended for production**（README），所有"实现选择"结论不得当语言契约。
