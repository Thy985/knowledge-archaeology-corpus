# 03 · Knowledge Layer — Aigis Generalized Knowledge（窄尖顶）

> L3/L4 抽象层。每个 KO 声明 `aggregation_rule`（R1-R4）+ 簇内 EK + 可回溯。升维是特例非默认——以下 7 个 KO 均有跨实例证据；未跨项目验证的标 `Cross-project validation pending`。

## KO-01: 编码/变体攻击的防御必须"先还原再匹配"，且还原产物要重新过全量检测

- **层**：L3 Pattern
- **aggregation_rule**：R1 机制簇（EK-02 mechanism↔EK-03 + EK-01 因果链）
- **簇内 EK**：EK-01、EK-02、EK-03、EK-04
- **内容**：对编码/混淆攻击（Base64/Hex/ROT13/零宽/Unicode tag），单独加"编码 pattern"不可靠；正确模式是 (a) 归一化击穿字符级绕过（NFKC/零宽/Confusable/Emoji），(b) 主动解码出变体，(c) 变体重新跑全量 pattern（跳过已命中 rule 防重复计分）。Aigis 实测：零宽字符攻击直接 100 命中，base64 攻击命中 `pi_ignore_instructions (decoded)`。
- **解释范围扩大**：从"本项目特定绕过"到"任何基于文本的检测器对抗编码攻击的通用结构"；同类对照：WAF 的解码层、杀软的 unpack 层。
- **验证状态**：Aigis 内 S4（255 测试通过 + 实测矩阵）；**Cross-project validation pending**（未在第二项目验证）。
- **可回溯**：EK-01/02/03/04 → aigis/scanner.py / decoders.py / 实测记录。

## KO-02: 信任分层是代理安全的基础结构——外层权限门 + 内层策略钩子 + 审计第三轨

- **层**：L4 Cognitive Model
- **aggregation_rule**：R2 因果链簇（EK-06→EK-08→EK-18，沿 causal/constraint 边）
- **簇内 EK**：EK-06、EK-07、EK-08、EK-09、EK-17、EK-18、EK-19
- **内容**：代理工具调用防御存在两个独立门控层：**宿主自身的静态权限表**（Claude Code deny/ask/allow，独立于 hook 返回生效，是外层门）与**代理框架的 hook/拦截层**（是内层门，能看到宿主门放行后的调用）。策略必须显式区分"能在宿主权限表表达的规则"（静态、无条件）与"只能在 hook 层评估的规则"（带条件/无工具等价物），并在导出时拒绝静默降级（条件规则不可导出，宁排除保留在 hook 层）。审计是第三轨：允许失败而不改变决策，防止审计能力拖垮安全关键路径。
- **解释范围扩大**：从 Aigis+Claude Code 组合到"任何宿主+插件双层代理架构"（如 IDE 扩展 + agent 框架、浏览器插件 + 自动化工具）；一句话稳定关系：**宿主门与钩子门的职责必须显式划分，条件规则只能活在能评估条件的层**。
- **验证状态**：Aigis 内 S4（settings_export 测试 + hook 测试）；**Cross-project validation pending**。
- **可回溯**：EK-08（ExcludedRule 机制）→ settings_export.py；EK-18 → claude_code.py HOOK_SCRIPT。

## KO-03: 数据流污点（taint）比授权（capability）更早做安全判定——控制流工具永远不吃不可信数据

- **层**：L4 Cognitive Model
- **aggregation_rule**：R3 不变量簇（EK-10/EK-11 constraint 汇聚 + EK-12/EK-13/EK-14 subsystem）
- **簇内 EK**：EK-10、EK-11、EK-12、EK-13、EK-14
- **内容**：CaMeL 分离（arXiv 2503.18813）的核心不变量：**数据来源（taint）先于能力授权（capability）判定**——UNTRUSTED 数据（工具输出/外部 API/RAG）驱动控制流工具（shell/agent spawn/eval/MCP 调用）时无条件拒绝，**即使存在匹配的 capability grant**；taint 提升不可跳级（UNTRUSTED 必须经扫描 SANITIZED 才能 TRUSTED）。这是"间接注入→任意代码执行"这条攻击链的结构性断路。
- **解释范围扩大**：从 Aigis 到所有 agent 工具调用系统（OpenAI function calling / LangChain tools / MCP client）；一句话稳定关系：**不可信数据能做什么，不由它被授权了什么决定，而由它从哪来决定**。
- **验证状态**：实现 S3（enforcer 代码存在）+ 文档 S2；**测试覆盖缺失**（EK-14 观察：tests/ 无 capabilities 引用）——故本 KO 标记 `Partially validated (implementation only)`，升维谨慎。
- **可回溯**：EK-10 → enforcer.py（_CONTROL_FLOW_RESOURCES + 无条件 DENY 分支）；EK-11 → taint.py。

## KO-04: 防篡改审计 = 签名 + 哈希链双机制，且审计失败不得反噬决策

- **层**：L3 Pattern
- **aggregation_rule**：R1 机制簇（EK-15 mechanism↔EK-16，EK-17 contrast↔EK-18）
- **簇内 EK**：EK-15、EK-16、EK-17、EK-18、EK-19
- **内容**：可提交给安全审查的审计日志需要：逐条 HMAC 签名（防外部篡改）+ 覆盖**含签名全字段**的哈希链（防"重签整链"内部篡改）+ genesis 锚点；同时审计路径与决策路径**显式解耦**（日志失败不改变 allow/deny），安全关键路径（hook 的解析/导入/扫描/策略）则 fail-closed。两套失败语义相反且都有测试锚定。
- **解释范围扩大**：从 Aigis 到任何"可验证性要求 + 运行时防御"系统（供应链 attestation、合规记录）；同类对照：区块链 hash chain、git 对象链、Sigstore。
- **验证状态**：S4（54 审计测试通过，篡改 3 字段均验签失败）。
- **可回溯**：EK-15/16 → audit/{signed_log,chain}.py；EK-17/18 → claude_code.py + test_signed_audit_hook.py。

## KO-05: "诚实性设计"是安全产品可信度的一部分——不利数据照实呈现、自有编号明示非官方

- **层**：L3 Pattern
- **aggregation_rule**：R4 主题簇（EK-28/EK-29/EK-30/EK-31/EK-33 覆盖合规、测量、基准互补维度）
- **簇内 EK**：EK-28、EK-29、EK-30、EK-31、EK-33
- **内容**：Aigis 在多处把"审阅者能验证"设计进产物：合规映射的 GL-*/SEC-*/APPI-* 字段注释明示是**自有编号、非官方条款号**（审阅者无法在指南中查证）；trust pack 声明"从实时配置生成、非营销声明"；OSS 对比 benchmark 页声明"对 Aigis 不利的类别表格照样承认"；ROADMAP 每个数字标 measured。定位统一为"supporting evidence，非合规认证"。
- **解释范围扩大**：从 Aigis 到任何面向安全审查/审计的安全产品——被审阅方主动暴露证据边界能降低审查摩擦；同类对照：SBOM 的 SPDX 依赖声明、红队报告的失败案例保留。
- **验证状态**：S3（代码/文档中实现）；Cross-project validation pending。
- **可回溯**：EK-28 → trust_pack.py ControlMapping 注释；EK-30 → docs/benchmarks/oss-comparison.md；EK-31 → ROADMAP.md。

## KO-06: 战略测量的纪律——放弃错误的 KPI 比加码更正确

- **层**：L4 Cognitive Model（项目级 → 可迁移）
- **aggregation_rule**：R2 因果链簇（EK-31→EK-32 causal + EK-33）
- **簇内 EK**：EK-31、EK-32、EK-33、EK-35
- **内容**：Aigis 用实测数据否决了 1000-star 目标（54 实际、增长 ~4/月）与检测数竞赛（768 vs 260+、headcount 输局），并**明示退出清单**（pattern 数、通用货架、英文市场、SaaS 路线图）与 tradeoff。关键可迁移命题：**当"获胜渠道不产生目标 KPI"时（300 点赞不转化 stars），错误的 KPI 会让整个路线图漂移；结构性缺位（日本部委指南没有映射工具）比红海竞争更值得占据**。
- **解释范围扩大**：从 Aigis 到所有开源/单人维护项目的方向决策；一句话稳定关系：**测量先于方向，退出清单与目标清单同样重要**。
- **验证状态**：项目内 S3（ROADMAP 文档）；**Cross-project validation pending**（开源项目战略文献支持但未在第二项目实证）。
- **可回溯**：EK-31/32/33 → ROADMAP.md。

## KO-07: 工程时序硬约束防级联烧号——"先落地再打标、撞号不重试"

- **层**：L3 Pattern
- **aggregation_rule**：R3 不变量簇（EK-34/EK-35 constraint 汇聚到发布不变量）
- **簇内 EK**：EK-34、EK-35
- **内容**：版本号/标签是外部状态（PyPI/SemVer），一旦烧号（v2.0.0、v1.1.1→v1.1.2→v1.1.3 级联）不可逆；防御 = 发布流程硬约束（tag 必须 reachable from master）+ 前置预检（tag 前查外部注册表）+ 明确的反模式清单（tag collision 时禁止 bump 版本从同一孤儿 commit 重试）。
- **解释范围扩大**：从 Aigis 到任何发布到外部注册表的项目（PyPI/npm/Go proxy/Docker Hub）；同类对照：npm publish 幂等、Docker tag 覆盖策略。
- **验证状态**：S4（test_release_preflight + release.yml orphan-tag 保护）。
- **可回溯**：EK-34 → CLAUDE.md + release.yml；EK-35 → CHANGELOG.md + preflight 脚本。

---

## KO 聚合规则矩阵

| KO | 规则 | 簇内 EK | 边类型 | 解释范围扩大论证 |
|----|------|---------|--------|-----------------|
| KO-01 | R1 机制簇 | EK-01/02/03/04 | mechanism+causal | 编码对抗通用结构 |
| KO-02 | R2 因果链簇 | EK-06/07/08/09/17/18/19 | causal+constraint | 宿主+插件双层架构 |
| KO-03 | R3 不变量簇 | EK-10/11/12/13/14 | constraint | agent 工具调用通用 |
| KO-04 | R1 机制簇 | EK-15/16/17/18/19 | mechanism+contrast | 可验证性+运行时防御 |
| KO-05 | R4 主题簇 | EK-28/29/30/31/33 | subsystem 互补 | 安全产品可信度 |
| KO-06 | R2 因果链簇 | EK-31/32/33/35 | causal | 开源方向决策 |
| KO-07 | R3 不变量簇 | EK-34/35 | constraint | 外部注册表发布 |

## 配比健康度（对照 v3 目标）

- Facts/Evidence：100+（snapshot 中结构化事实约 110+）
- EK：37（40-60 目标下限附近，小项目合理缩放）
- Pattern/KO：7（7-12 目标下限）
- 无"同子系统=聚合理由"假聚合；所有 KO 可回溯 EK/证据。
