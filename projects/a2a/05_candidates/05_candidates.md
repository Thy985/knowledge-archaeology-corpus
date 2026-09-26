# 05 — Candidates（未验证假说 / 跨项目假说 / 不确定结论）

> 全部为 Hypothesis（非 Fact/Pattern）。不因结论漂亮就升层；Cross-project Candidate 不写成已验证 Principle。

## C-01 跨项目假说：任务不可变性 + 会话延续的跨协议同构
- **假说**：A2A 的"终态不可变 + contextId 会话延续"与 E2B 的 session 生命周期、Codex harness 的 run 不可变模型共享同一认知结构——"执行单元不可变、会话上下文延续"
- **当前证据**：A2A `[T:life §Task Immutability]`；E2B 考古（corpus projects/e2b）session 模型
- **缺失证据**：至少一个第三方项目独立复现该结构（当前仅 2 个来源，未达 S8）
- **验证路径**：考古 hermes-agent（corpus PR#19 待合并）或 dynamo 时比对 session/task 语义
- **验证所需**：S8 Repeated/Cross-project

## C-02 跨项目假说：Agent 安全"fail-closed 扩展边界"是通用模式
- **假说**：A2A"扩展不得绕过主安全控制"、E2B 沙箱网络面 fail-closed、Tafcm 受控接口（受控工具+权限矩阵）是同一 Agent 安全原则的三个实例：**新增能力面必须与主安全控制同等级约束**
- **当前证据**：A2A `[T:ext §Security]`；E2B（沙箱 fail-closed）；Tafcm（受控接口，skill 历史验证）
- **缺失证据**：更广的第三方 Agent 项目（非 sandbox/协议类）独立确认
- **验证路径**：后续考古 browser-use / dynamo 时检查其权限模型
- **验证所需**：S8

## C-03 路由哲学对比假说：不透明 tenant vs 声明式 DID 路由
- **假说**：A2A 的"不透明 tenant + 客户端回显义务"（`[T:multi]`）与 Agent Mesh / DID 式可解析身份路由是两种相反哲学——前者把路由语义完全留在服务器（协议只定义义务），后者把身份解析进协议。A2A 的取舍换来了协议简洁，代价是跨运营商路由不可互操作
- **当前证据**：A2A 文档 `[T:multi]`（语义在 server）
- **缺失证据**：Agent Mesh 等系统的对比实测；A2A 社区是否出现 registry 标准化（`[T:disco §Future Considerations]`"community explores standardizing registry interactions"）
- **验证路径**：横向调研 Agent Mesh / 微软 Agent Governance Toolkit

## C-04 协议仓库 CI 治理模式可迁移性
- **假说**：无测试体系协议仓库用"lint/规范一致性/链接检查/发布门"作为 CI 主体（`.github/workflows/` 11 个 workflow）可保证规范质量——此模式适用于协议/规范类仓库，但不适用于运行时实现
- **当前证据**：A2A 仓库本身（`[.github/workflows]` 实证）
- **缺失证据**：另一个规范仓库（如 MCP spec）的 CI 结构对比；协议 bug 率数据
- **验证路径**：考古 MCP 仓库或 OpenAPI spec 仓库时对比
- **不确定性**：本 run 无法观测 CI 实际运行结果（只看到 workflow 定义），行为有效性为推断

## C-05 推送通知 SSRF 防护与 OWASP Agentic 安全映射
- **假说**：A2A 的 webhook URL 验证三件套（allowlist/ownership verification/egress firewall，`[T:stream]`）与 OWASP/行业 Agent 安全建议（防 SSRF、防 prompt 注入外扩）是同一防御谱系；A2A 是少见的在协议规范层面显式写 SSRF 义务的协议
- **当前证据**：`[T:stream §Security Considerations]`
- **缺失证据**：业界规范对比 + 真实攻击案例统计
- **验证路径**：检索 OWASP Agentic Security 与 LLM 供应链攻击文献

## C-06 ProtoJSON enum 大小写 UX 成本 vs 标准化收益
- **假说**：ADR-001 选择 ProtoJSON 后，enum SCREAMING_SNAKE_CASE 的破坏性变更与开发者困惑（ADR 自列"Ugly enums"）是否会转化为实际实现错误率——标准化收益是否真实大于 UX 成本
- **当前证据**：`[ADR]` 决策记录（明确列出负面）
- **缺失证据**：实现者反馈/错误率数据；SDK 是否已消化该成本
- **验证路径**：a2a-sdk 实现与 issue 检索

## C-07 扩展 metadata 叠加 vs MCP Skills 治理的关系
- **假说**：A2A"扩展只允许 metadata 叠加，不改核心类型"（`[T:ext §Limitations]`）与 2026-09-24 雷达观察的 "SEP-2640 Skills over MCP Final"（MCP skills 治理）是同一趋势的两端：**协议生态都在把"能力扩展"从核心契约移到受控的附加层**
- **当前证据**：A2A `[T:ext]`；KnowlegeMap 雷达日志 2026-09-24（SEP-2640 条目）
- **缺失证据**：SEP-2640 全文与 A2A 治理的逐条对比
- **验证路径**：读取 SEP-2640 规范；对比两者激活/晋升机制
- **验证所需**：跨协议横向（C-07 为高价值 Benchmark 候选）

---

## C-08 错误映射/错误面研究（Validation MISSING-2 补录）
- **假说**：规范 §5.4 Error Code Mappings 的错误类型表（如 `TaskNotFoundError`/`UnsupportedOperationError`，见 `[T:cpb]` 引用）构成协议级"错误面"——错误类型与 HTTP/gRPC 状态码的映射方式会影响实现质量与互操作失败模式，值得作为独立知识单元展开
- **当前证据**：`[T:cpb §Error Mapping]`（要求自定义绑定提供等价映射表）；proto 内错误类型注释
- **缺失证据**：完整错误类型枚举清单与各绑定映射表（本 run 未展开）
- **验证路径**：refresh run 中读 specification §5.4 + SDK 错误处理实现

## Candidates 与 KO 区分声明
- C-01/C-02 是对 KO-01/KO-04 的**跨项目验证诉求**（KO 为单项目 Pattern，Candidates 为其泛化未验证形态）
- C-03~C-07 为**无单项目完整证据链**的探索性假说（部分有单源文档证据，但未达 Pattern 成立条件：内聚/跨实例/解释范围/可命名/可回溯五条件缺跨实例）
