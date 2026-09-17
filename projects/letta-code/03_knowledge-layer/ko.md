# 03 Knowledge Layer — Generalized KO（9 个）

> 每个 KO 必须声明 aggregation_rule（R1 机制簇 / R2 因果链簇 / R3 不变量簇 / R4 主题簇）+ 簇内 EK + 升维论证（L1→L5）。严格区分 Fact/Observation/Hypothesis/Pattern/Cognitive Model/Principle。

---

## KO-01 记忆即文件系统（MemFS Pattern）— L3 Pattern
- **aggregation_rule**: R1 机制簇（EK-01/02/03/05/06/08 共享 mechanism 边，跨记忆子系统与工具子系统两个独立文件族）
- **KO 陈述**: 把 agent 记忆建模为**版本化文件树**（git repo + Markdown + frontmatter），记忆写操作变成受审计的文件变更，检索变成投影 + 延迟读取——记忆获得与代码**同构**的版本化审计、可回滚性与工具生态（Reconciliation 修正：原"同等"绝对化措辞收紧为"同构"）。
- **簇内 EK**: EK-01（git 仓库）、EK-02（生命周期）、EK-03（hook 门禁）、EK-05（投影）、EK-06（reason 留痕）、EK-08（渐进披露）
- **解释范围扩大论证**: 从"Letta 如何存记忆"扩展到"任何需要持久 agent 状态的系统如何设计记忆层"——OKF/Anthropic /mnt/memory/Grok Build 的"记忆=文件"路线同构（跨项目候选，见 candidates）。
- **反例预算**: ①mem0/MemGraphRAG 用向量库而非文件树（抽取式）；②记忆不用 git 的 agent 也能工作（纯内存/DB）；③文件树投影有 context 成本，超大树会退化。
- **L1 证据**: F-21/F-23/F-24/F-27/F-39

## KO-02 fail-closed 记忆治理（Memory Confinement Principle）— L4 Cognitive Model
- **aggregation_rule**: R3 不变量簇（EK-12/13/14/15/17 经 constraint/dependency 边汇聚到"无审批回退路径必须内核强制"这一安全不变量）
- **KO 陈述**: **可回退路径（人工审批）与不可回退路径（unattended 子代理）必须采用不同的信任基线**——后者必须 fail-closed（无沙箱即抛错），前者可 opt-in。
- **升维链**: L1（EK-13 抛错）→ L2（因为非交互无 approve/deny）→ L3（unattended 执行面必须默认最小权限）→ L4（信任基线由"是否有人工回退"决定，不由"是否重要"决定）
- **反例预算**: ①sandbox-gate 交互路径 opt-in 存在（对照而非反例，说明双基线是有意的）；②`LETTA_FS_SANDBOX=0` 可关闭——显式退出开关是反例信号；③若某平台 sandbox 永不可用，记忆子代理完全不可用（fail-closed 的代价）。
- **L1 证据**: F-46/F-48/F-49/F-80

## KO-03 记忆权限 = 路径模型（Path-as-Policy）— L3 Pattern
- **aggregation_rule**: R1 机制簇（EK-12/17/44 + EK-43 经 mechanism 边：隔离、顺序语义、双树、环境变量契约共享"路径即策略"机制）
- **KO 陈述**: 跨 agent 记忆隔离通过**文件系统路径墙**实现而非权限列表——deny 整个 agents 树、carve 自我目录、顺序表达特异性；策略与沙箱同源，避免策略漂移。
- **反例预算**: ①路径墙无法防"agent 通过工具逃逸"（需内核沙箱兜底，EK-13 正是此依赖）；②Windows 路径大小写/分隔符差异（windows-credentials 测试存在）；③符号链接逃逸未在本快照验证（candidate）。
- **L1 证据**: F-47/F-78/F-22/F-88

## KO-04 自编辑记忆 + 提交理由（Self-Editing Memory with Rationale）— L3 Pattern
- **aggregation_rule**: R4 主题簇（EK-04/06/07/10/11 覆盖"记忆如何被 agent 自己修改"主题的互补维度：格式、工具、重组、迁移、共享）
- **KO 陈述**: agent 记忆自编辑必须具备**结构化工具 + 强制理由 + 写入门禁**三件套——工具约束操作原语，reason 让变更可解释，hook 校验内容合法。
- **反例预算**: ①让 agent 直接写文件（系统提示词允许"直接编辑投影文件"——v2 同步指令）是无工具路径，但 hook 仍校验；②reason 可能被 agent 编造（内容真实性无验证）；③legacy read_only 与 v2 白名单语义不一致（D1 争议）。
- **L1 证据**: F-29/F-34/F-35/F-39/F-40

## KO-05 渐进披露记忆索引（Progressive Disclosure Index）— L3 Pattern
- **aggregation_rule**: R4 主题簇（EK-08/09/05 覆盖"记忆如何在上下文中呈现"主题：索引、预算、投影）
- **KO 陈述**: 记忆分两级披露——恒驻核心（顶层 .md 投影进系统提示词）+ 延迟加载（子目录经 MEMORY.md 索引按需读取）；用目录存在性作为"这是记忆"的标记，避免全量上下文。
- **反例预算**: ①core memory 全部投影有 size 预算（maxCoreMemoryCharacters），超限怎么办（约束在，行为未验证）；②索引被删则子树不可达（隐式契约）；③向量检索式记忆不做分级披露（memgraphrag 对照）。
- **L1 证据**: F-31/F-32/F-33/F-36/F-37

## KO-06 双运行时一致性契约（Dev/Prod Parity）— L3 Pattern
- **aggregation_rule**: R4 主题簇（EK-24/25/34/36/37 覆盖"工程如何保证行为一致与可维护"）
- **KO 陈述**: 源码运行时（Bun）与发布运行时（Node bundle）的行为差异必须被显式识别并双路径测试；工程纪律（@/、kebab-case、named exports、mock 隔离）服务于**可搜索性与可验证性**而非风格偏好。
- **反例预算**: ①4 文件豁免 @/ 规则说明规则总有例外；②lint 只是 CI 门之一，不保证运行时行为；③"agent 可搜索性"论点在其他语言生态（Python）不直接适用。
- **L1 证据**: F-57/F-58/F-71/F-72/F-116

## KO-07 治理双轨：权限模式 vs 沙箱强制（Governance Dual-Track）— L4 Cognitive Model
- **aggregation_rule**: R3 不变量簇（EK-18/19/20/21/22 + EK-13/14 经 constraint 边汇聚到"执行权必须分层"不变量）
- **KO 陈述**: **决策权（权限模式）与执行约束（沙箱/内核策略）是两条正交治理轨**——权限模式决定"agent 能否请求"，沙箱决定"进程物理上能做什么"；高安全路径（记忆子代理）跳过决策轨直接内核强制。
- **升维链**: L1（四模式 + 双沙箱）→ L2（交互需审批故 opt-in，非交互 fail-closed）→ L3（决策/执行分离模式）→ L4（权限是协商层，沙箱是物理层）
- **反例预算**: ①DEFAULT_PERMISSION_MODE=unrestricted 表明默认协商层最宽（D2 争议）；②mods 能力面可全开/裁剪，是第三条轨；③内核沙箱不可用时 fail-closed 直接禁功能。
- **L1 证据**: F-17/F-18/F-46/F-76/F-79

## KO-08 工程化记忆系统的方法论（Memory-as-System Methodology）— L5 Methodology
- **aggregation_rule**: R2 因果链簇（EK-01→EK-05→EK-06→EK-02→EK-03→EK-10 完整链：存储→投影→自编辑→提交→门禁→迁移）
- **KO 陈述**: 构建 agent 记忆系统的可操作准则（**建议准则，非普适定律**——Reconciliation 修正措辞）：①记忆必须是版本化资产（git）；②写入必须有理由与门禁（reason + hook，软门禁）；③披露必须分级（core 投影/延迟读取）；④演化必须显式（reflection/迁移）；⑤隔离必须物理（内核沙箱）。
- **反例预算**: ①小 agent 可无 git 记忆（过度工程风险）；②"reason 必填"在高速工具循环中增加延迟；③方法论源自单项目（letta-code），跨项目验证 pending。
- **L1 证据**: 整条因果链 EK 引用
- **注**: 跨项目验证 pending——这是本项目导出的候选方法论，不是已验证的普适定律（Epistemic: Methodology, single-source）。

## KO-09 记忆 = 决策留痕（Memory as Decision Trace）— L3 Pattern
- **aggregation_rule**: R2 因果链簇（EK-06→EK-02→EK-03：reason 必填→commit 历史→hook 校验，形成"为什么改记忆"的可审计链）
- **KO 陈述**: 每次记忆变更强制 reason + git commit，使"agent 为什么改变自己的认知"成为可审计记录——记忆演化史 = 决策轨迹。
- **反例预算**: ①reason 无真实性验证（agent 可写虚假理由）；②post-turn push 失败时本地 commit 与远端不一致（retry 存在但最终一致性未验证）；③记忆合并（reflection auto-merge）时 reason 由子代理生成，来源可信度存疑。
- **L1 证据**: F-29/F-39/F-25

---

## KO 聚合规则矩阵
| KO | rule | 簇内 EK | 边类型 | 解释范围扩大 |
|---|---|---|---|---|
| KO-01 | R1 | 01/02/03/05/06/08 | mechanism | 项目→agent 记忆层设计 |
| KO-02 | R3 | 12/13/14/15/17 | constraint/dependency | 记忆子代理→所有 unattended 执行面 |
| KO-03 | R1 | 12/17/43/44 | mechanism | 记忆隔离→沙箱策略设计 |
| KO-04 | R4 | 04/06/07/10/11 | 主题 | 记忆自编辑→agent 工具设计 |
| KO-05 | R4 | 05/08/09 | 主题 | 记忆呈现→上下文工程 |
| KO-06 | R4 | 24/25/34/36/37 | 主题 | 工程纪律→agent 代码库维护 |
| KO-07 | R3 | 13/14/18/19/20/21/22 | constraint | 治理轨→agent 运行时安全 |
| KO-08 | R2 | 01→05→06→02→03→10 | causal | 单项目→方法论（pending） |
| KO-09 | R2 | 06→02→03 | causal | 记忆变更→审计性设计 |

## 升维门状态（Abstraction Promotion Gate）
- L3 升维：KO-01/03/04/05/06/09 通过（可复用模板 + 同类对照成立）
- L4 升维：KO-02/07 通过（稳定关系一句话可述）
- L5 升维：KO-08 有条件通过（标注 cross-project pending，不冒充已验证法则）
- 降级记录：无（auditor 未判降级项到 KO 层；见 independent-audit）
