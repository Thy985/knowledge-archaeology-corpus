# 03 — Knowledge Layer（Generalized KO，窄尖顶）

> v3.1：每个 KO 声明 aggregation_rule（R1-R4）+ 簇内 EK + 边类型 + 解释范围扩大论证。跨项目数据点：aigis（ARCH-2026-09-13-001）+ guardian（ARCH-2026-09-14-001），**基线 PR #13/#14 open 未合并，引用来自本地 package**。

## KO 聚合规则矩阵

| KO | 层 | 聚合规则 | 簇内 EK | 边 | 证据强度 |
|---|---|---|---|---|---|
| KO-01 | L3 Pattern | R4 主题簇 | EK-01/02/03/04/13/29 | subsystem+causal | S3+S4+S8(3 项目) |
| KO-02 | L3 Pattern | R2 因果链簇 | EK-01→02→04→16 | causal | S3+S4+S8(2 项目) |
| KO-03 | L4 Cognitive Model | R3 不变量簇 | EK-05/08/13/20/28 | mechanism+contrast | S3+S8(2 项目) |
| KO-04 | L3 Pattern | R1 机制簇 | EK-06/07/12 | mechanism+constraint | S3+S4 |
| KO-05 | L4 Cognitive Model | R4 主题簇 | EK-16/17/18/19/32 | subsystem+dependency | S3+S4 |
| KO-06 | L4 Cognitive Model | R3 不变量簇 | EK-25/26/27 | dependency+constraint | S7 |
| KO-07 | L3 Pattern | R2 因果链簇 | EK-20/21/24/34 | causal | S3+S7 |
| KO-08 | L3 Pattern | R4 主题簇 | EK-14/15/17 | subsystem | S3+S4 |
| KO-09 | L3 Pattern | R1 机制簇 | EK-01/13/31 | mechanism | S3+S4+S8(3 项目) |
| KO-10 | L5 Methodology | R2 因果链簇 | EK-25/26/27/23 | causal+contrast | S7（跨项目验证中） |

---

## KO-01（L3 Pattern）确定性安全判定可审计

- **陈述**：Agent 安全网关把"判定"做成确定性的、可枚举的、可解释的管线（分层检测 → 显式权重 → 单点决策），而非让模型自我判断。
- **聚合规则**：R4 主题簇——EK-01（LangGraph 管线）/EK-03（加权聚合）/EK-04（决策优先级）/EK-13（并行扫描）围绕"确定性判定"互补。
- **跨项目**：Aigis L1-L7 检测管线 + HMAC 审计链（第 1 数据点）；Guardian 7 项确定性检查 + SHA-256 链（第 2 数据点）；AI Protector 7 层加权 + 三态（第 3 数据点）。**N-project corroborated = 3**。
- **解释范围扩大**：从"单项目有规则"扩大到"安全网关品类的通用架构模板"。

## KO-02（L3 Pattern）三态响应分级

- **陈述**：安全网关对攻击的响应不是二元的阻断/放行，而是分级动作——阻断、降级（改写/沙箱）、放行但记录。
- **聚合规则**：R2 因果链簇——EK-01→EK-02（管线→三态路由）→EK-04（决策）→EK-16（agent 门）。AI Protector 的 MODIFY（PII mask→transform）是**第三种响应**：保留请求但改写内容。
- **跨项目**：Guardian halt/sandbox/log-only（第 1 数据点，2 项目 corroborated）；对比 Aigis allow/deny/review。**数对项目数：三态分级目前 2 项目 corroborated**。
- **解释范围扩大**：响应分级是安全网关区别于"内容过滤器"的关键能力。

## KO-03（L4 Cognitive Model）规则层容错：检测层错误不得击穿热路径

- **陈述**：安全管线的检测层必须是可失败的——任何单层（含第三方模型/库）的故障、误报、缺依赖都应降级为"贡献风险信号"或"跳过"，而不是让整个请求路径崩溃或全盘误杀。
- **聚合规则**：R3 不变量簇——EK-05（NeMo 硬阻断→软贡献）/EK-08（A2 fail-safe）/EK-13（扫描器单失败不崩）/EK-20（阈值热更新）/EK-28（errors[] 累积）。共同不变量：**检测错误是可控成本，不是系统失败**。
- **跨项目**：Guardian check6 坏规则跳过（第 1 数据点，2 项目 corroborated）——**同构模式**：规则层错不崩热路径。
- **解释范围扩大**：从"具体 NeMo 误报修复"扩大为"安全热路径的通用健壮性不变量"。

## KO-04（L3 Pattern）检测视图与生成视图分离

- **陈述**：把"要检测的文本"（含解码变体）与"要送给 LLM 的文本"分离为两个视图——检测视图可任意变换，生成视图保持原文，LLM 永不见检测中间物。
- **聚合规则**：R1 机制簇——EK-06（build_scan_text）/EK-07（原文第一实验）/EK-12（scan_text 隔离）共享 A2 机制。
- **跨项目**：Guardian 无对应 deobfuscation 层（regex 静态匹配原文）——**contrast 对照**：AI Protector 是品类内首个显式反混淆管线。
- **解释范围扩大**：反混淆作为检测前置层可迁移到任何基于文本匹配的防护系统。

## KO-05（L4 Cognitive Model）内容防线与动作防线分离

- **陈述**：Agent 安全必须分两层：内容层（模型说什么——prompt/response 检测）与动作层（模型做什么——工具调用授权）。只做一层是"内容过滤器"，两层都做才是"agent 安全运行时"。
- **聚合规则**：R4 主题簇——EK-16/17（pre/post 双门）+EK-18（RBAC）+EK-19（确认暂停）+EK-32（威胁模型：响应与工具输出均不可信）。
- **跨项目**：Aigis/Guardian 均聚焦内容/请求层；AI Protector 的 RBAC 双门是**动作层的显式实现**（Guardian JWT ACL 为代理级授权，AI Protector 为 agent 内嵌工具级）。
- **解释范围扩大**：回应 README 核心命题——"Agent security is not about what the model says. It is about what the model does."

## KO-06（L4 Cognitive Model）可证明性：benchmark 是一等公民，不是事后报告

- **陈述**：安全工具的价值必须能被证明——把 benchmark 建成产品循环的一环（Find→Protect→Prove），用客观真值（planted canary/secret 精确匹配，no LLM-as-judge）而非模型自评来校准判定。
- **聚合规则**：R3 不变量簇——EK-25（grader 校准 94%→99%）/EK-26（6 数据集归一化）/EK-27（HARM_ML_MODE 权衡）。共同不变量：**不可验证的分数不是证明**。
- **主题呼应**：09-15 雷达 Proof-of-Guardrail（TEE 签名 attestation，arXiv 2603.05786）——同为"可证明安全"主题，但 AI Protector 走"benchmark 客观真值"路径而非 TEE 硬件路径。**Cross-project hypothesis（与 TEE 路径的对照）→ Candidates**。
- **解释范围扩大**：从"我们声称 99%"扩大到"安全声明必须带可复现的验证协议"。

## KO-07（L3 Pattern）部署可靠性工程：消除冷启动与隐性依赖

- **陈述**：含本地 ML 模型的安全服务，其生产可运行性由三类工程细节决定：模型预加载（冷启动）、依赖显式声明（隐藏运行时依赖）、阈值可热更新（免重启调优）。
- **聚合规则**：R2 因果链簇——EK-21（预加载 50s→0.9s）→EK-24（NeMo 低估依赖修复）→EK-20（阈值热更新）→EK-34（668 测试实测）。
- **解释范围扩大**：任何"本地模型 + 网关"架构都适用（对比 Guardian 无 ML 无冷启动问题——contrast）。

## KO-08（L3 Pattern）输出侧防线：LLM 响应不可信

- **陈述**：安全管线必须覆盖输出方向——响应中的 PII/secret/system-prompt 泄露在返回用户前被过滤；工具输出同样不可信（间接注入检测）。
- **聚合规则**：R4 主题簇——EK-14（三防线）/EK-15（system-leak 局限）/EK-17（post-tool 间接注入）。
- **跨项目**：Aigis exfil 检测为部分对应；Guardian **无输出侧**——AI Protector 是品类内输出侧最完整的（PII/secrets/system-leak 三线）。
- **解释范围扩大**：输入侧检测完备后，输出侧是剩余攻击面（系统提示泄露、间接注入）。

## KO-09（L3 Pattern）图编排 + 并行扫描的网关架构

- **陈述**：用状态图（LangGraph StateGraph）显式编排检测管线，扫描器 asyncio 并行执行——管线结构可读、可扩展、单节点可替换。
- **聚合规则**：R1 机制簇——EK-01（9 节点图）/EK-13（并行扫描）/EK-31（agent 11 节点图）共享图编排机制。
- **跨项目**：Aigis/Guardian 均为顺序 fail-fast 管线——**AI Protector 是品类内首个图编排 + 并行扫描**。3 项目 corroborated（编排方式对比）。
- **解释范围扩大**：从"顺序检查列表"到"可路由状态图"是安全管线工程化的演进方向。

## KO-10（L5 Methodology）先测后防再证（Find → Protect → Prove）

- **陈述**：落地 agent 安全的标准动作序列：先用可复现 benchmark 测出真实缺口（Find）→ 在调用路径上插入确定性强制（Protect）→ 用客观真值 grader 证明守住（Prove），并循环（re-scan）。
- **聚合规则**：R2 因果链簇——EK-25/26/27（benchmark 基建）→EK-23（已知绕过驱动测试迭代）→EK-07（实验驱动检测设计）。
- **状态**：**Methodology（ai-protector-originated · Strongly evidenced · Cross-project validation pending）**——单项目强证据，跨项目验证待定。
- **解释范围扩大**：可操作准则：①安全声明必须绑定可复现 benchmark；②判定可信度要区分 mechanical/exact 与 heuristic；③任何"新增检测"先跑回归场景再上线。

---

## 三层配比检查

| 层 | 目标 | 实际 |
|---|---|---|
| Facts/Project Layer | 1 地图 | 01_project-layer.md ✓ |
| Engineering Knowledge | 40~60（本项目按规模缩放） | 36 条 EK，links 全覆盖 ✓ |
| Generalized KO | 7~12 | 10 个 KO（L3×6 / L4×3 / L5×1）✓ |
| Candidates | 未验证假设 | 05_candidates.md ✓ |

**升维纪律**：KO-06/KO-10 跨项目验证未完成，标 `Cross-project validation pending`；三态分级明确标注"2 项目 corroborated"（数对项目数）。
