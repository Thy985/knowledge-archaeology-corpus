# Wave 3 · Knowledge Objects（L1→L5 五层合成）

> 由 knowledge-synthesizer 在 Evidence Graph + Problem/Decision/Flow/Pattern Graph 上合成。
> 每个 Knowledge Object 按五层阶梯（L1 事实→L2 知识→L3 模式→L4 模型→L5 方法论）组织。
> 时间：2026-09-03

---

## KO-01 · Intelligence ≠ Authority（判断力与执行权分离）

| 字段 | 值 |
|------|-----|
| category | AGENT / PERMISSION / ARCHITECTURE |
| abstraction | L4 |
| value | A |
| epistemic_status | Validated Pattern（codex 项目内多源验证；跨项目验证 pending） |
| confidence | high |

### L1 工程事实
- ToolOrchestrator::run() 对任何工具调用驱动固定序列：Approval Gate（Skip/Forbidden/NeedsApproval 三态）→ Sandbox Selection（SandboxManager.select_initial）→ 首次尝试 → 沙箱拒绝时升级重试（重试无需重新审批，approval caching）（orchestrator.rs:125-527）
- approval_policy 三态：Never（Full Access，禁用沙箱时不审查直接批准）/ OnRequest（Guardian 独立 AI 审查自动批准）/ AlwaysAsk（始终请求用户）（orchestrator.rs:135, 170-230）
- Guardian 是独立的 review session，克隆父配置继承网络策略，90 秒超时，fail closed（guardian/mod.rs:1-13）
- 模型（产生 tool_call 的智能）不直接拥有执行权，执行权由 Approval + Sandbox + Network Proxy 三重 Gate 控制

### L2 工程知识
因为 Agent 判断不可靠（Guardian fail closed 设计本身就是对 AI 审查不可靠的承认），一个错误判断 = 一次未授权变更，所以将**判断力**（模型推理产生 tool_call）与**执行权**（审批+沙箱+网络 Gate）分离为两个独立维度。模型可以"建议"执行，但"是否执行"由独立的 Gate 链决定。

### L3 工程模式
职责分离 / 最小权限 / 受控接口在 Agent 系统的实例。
- 同类对照：lint/CI 静态分析（代码提交前的独立验证 Gate）、human-in-the-loop（高风险操作的人工审批 Gate）、sudo（权限提升的独立认证 Gate）
- codex 实例：模型（Intelligence）→ Approval Gate（Authority 第一层）→ Sandbox Gate（Authority 第二层）→ Network Proxy Gate（Authority 第三层）→ 执行

### L4 认知模型
**Intelligence ≠ Authority** —— 判断力（智能）与执行权（权威）是两个独立维度。智能系统的执行权必须独立于产生意图的智能，由可审计、可降级、可熔断的 Gate 链控制。智能越高，执行权的 Gate 应越严格，而非越宽松。

### L5 方法论
1. Agent 影响状态的动作必须受控接口化，不允许智能体直接调用系统调用
2. 证明与执行分离：产生意图的模块不拥有批准执行的权力
3. 权限按可逆性×影响授予：可逆低影响动作可自动批准，不可逆高影响动作必须人工或多 Gate
4. 即使 AI 全错也可回滚可验证可恢复：每个 Gate 的决策必须可审计
5. 执行权 Gate 链应支持降级（沙箱拒绝→无沙箱重试）但降级必须触发额外审批

### provenance
- discovered_by: code-analyst（EV-C003）+ authority-analyst（EV-A001, EV-A002）
- supported_by: test-analyst（EV-T002 集成测试覆盖）+ failure-analyst（EV-F004 fail closed 设计）
- contested_by: 无

### evidence
- supporting: EV-C003, EV-A001, EV-A002, EV-F004, SF-01, SF-02
- contradicting: 无

### scope
- applies_when: 任何拥有自主执行能力的 AI Agent 系统（CLI Agent / 桌面 Agent / 云 Agent）
- does_not_apply_when: 纯建议型 AI（无执行能力，仅输出文本）

### flows
- control: F-01（工具执行控制流）
- authority: F-05（工具执行权限流）
- evidence: F-04（Guardian 审批证据流）

---

## KO-02 · 声明 ≠ 证据（AI 审批的证据生产链）

| 字段 | 值 |
|------|-----|
| category | AGENT / EVALUATION / TESTING |
| abstraction | L4 |
| value | A |
| epistemic_status | Validated Pattern |
| confidence | high |

### L1 工程事实
- Guardian review 重建 compact transcript：message transcript ≤20K tokens / tool transcript ≤10K / 单条 message ≤5K / 单条 tool ≤1K / recent ≤40 条（guardian/mod.rs:71-77）
- Guardian 返回结构化 GuardianAssessment：risk_level / user_authorization / outcome / rationale（guardian/mod.rs:160-167）
- 90 秒超时，超时/执行失败/格式错误 → fail closed（默认拒绝）（guardian/mod.rs:62, 12-13）
- GuardianRejectionCircuitBreaker：连续 3 次或最近 50 次中 10 次拒绝 → InterruptTurn；CyberModel 策略更严格（连续 1 次即中断）（guardian/mod.rs:64-68, 196-232）
- 网络审批采用延迟确认：begin → ActiveNetworkApproval（含 proxy + cancellation_token）→ 执行 → DeferredNetworkApproval → finish（orchestrator.rs:66-123）

### L2 工程知识
因为 AI 审批（Guardian）本身也是 AI，同样可能出错，所以不能把"Guardian 说可以"当成"真的可以"。必须建立完整的证据生产链：上下文重建（transcript）→ 结构化评估（JSON）→ 超时保护 → 熔断器 → 延迟确认。每一步都是可审计的证据，而非简单的"批准/拒绝"声明。

### L3 工程模式
复现驱动验证模式 / 证据链完整性模式。
- 同类对照：TDD red→green（必须先看到失败才能承认修复）、bug 报告"可复现"门槛（不可复现 = 无效证据）、CI/CD 的部署门禁（必须通过所有检查才能部署）
- codex 实例：Guardian 评估不是"AI 看一眼说行"，而是有 token 预算、结构化输出、超时、熔断器的完整证据生产链

### L4 认知模型
**声明 ≠ 证据；信任 = 证据链的完整性与真实性。** AI 系统的任何自动决策（包括 AI 对 AI 的审批）都不能仅凭声明被信任，必须有完整的、可审计的、可熔断的证据生产链支撑。

### L5 方法论
1. 每次自动审批必须绑定可审计的评估记录（输入上下文 + 结构化输出 + 耗时）
2. 未通过评估（超时/格式错误/执行失败）不承认批准，默认拒绝
3. 不同动作允许不同证据强度：低风险动作可简化证据链，高风险动作必须完整
4. 自动审批系统必须有熔断器：连续拒绝达到阈值时中断执行，防止无限重试
5. 延迟确认模式：先放行执行但保持可取消（cancellation_token），执行后最终确认

### provenance
- discovered_by: authority-analyst（EV-A002, EV-A005）+ code-analyst（EV-C007）
- supported_by: failure-analyst（EV-F004）
- contested_by: 无

### evidence
- supporting: EV-A002, EV-A005, EV-C007, EV-F004, SF-02, SF-08
- contradicting: 无

### scope
- applies_when: AI 系统中存在自动审批/自动决策的场景
- does_not_apply_when: 纯人工审批场景

### flows
- evidence: F-04（Guardian 审批证据流）
- authority: F-05（网络延迟确认）

---

## KO-03 · 自举系统的递归矛盾与显式逃逸阀

| 字段 | 值 |
|------|-----|
| category | ARCHITECTURE / ENGINEERING / RUNTIME |
| abstraction | L3 |
| value | B |
| epistemic_status | Validated Pattern |
| confidence | medium |

### L1 工程事实
- CODEX_SANDBOX_NETWORK_DISABLED 和 CODEX_SANDBOX 环境变量由沙箱运行时设置，代码中用于提前退出无法在沙箱中运行的测试（如需要自己 spawn Seatbelt 的集成测试）（AGENTS.md:8-10）
- AGENTS.md 明确规定"Never add or modify any code related to CODEX_SANDBOX_NETWORK_DISABLED_ENV_VAR or CODEX_SANDBOX_ENV_VAR"（AGENTS.md:8）
- Agent 自身运行在沙箱中，但 Agent 的功能包括管理沙箱（配置沙箱策略、spawn 沙箱子进程），形成递归

### L2 工程知识
因为 Agent 自身运行在沙箱中，但 Agent 的功能包括管理沙箱，被管理的系统同时是管理者的宿主，形成自举递归。沙箱内无法运行需要自己 spawn 沙箱的测试，所以需要环境变量作为隐式契约来打破递归——运行时告诉代码"你现在在沙箱里，不要尝试嵌套沙箱"。

### L3 工程模式
自举系统的逃逸阀模式。
- 同类对照：编译器自举的 stage0（用其他语言写的最小编译器来编译自身）、虚拟机的 host-guest 通道（VM 内无法直接访问 host 硬件，需要虚拟设备作为逃逸阀）、Docker-in-Docker 的 socket 挂载（容器内控制容器需要挂载 docker.sock）
- codex 实例：CODEX_SANDBOX* 环境变量是沙箱自举的逃逸阀，由外部运行时设置，内部代码只读，禁止修改

### L4 认知模型
自举系统中，被管理者同时是管理者的宿主，必须存在显式逃逸阀来打破递归。逃逸阀必须由外部设置、内部只读，其存在本身应被文档化和治理（禁止修改）。

### L5 方法论
1. 设计自举系统时必须识别递归点（管理者同时是被管理者）
2. 为每个递归点提供显式逃逸阀（环境变量/配置文件/外部通道）
3. 逃逸阀必须由外部设置，禁止内部代码修改
4. 逃逸阀的存在和语义必须被文档化（如 AGENTS.md 中的明确规定）
5. 逃逸阀触发的行为（提前退出/降级模式）必须有测试覆盖

### provenance
- discovered_by: failure-analyst（EV-F001）+ doc-analyst（EV-D001）
- supported_by: code-analyst（EV-C001 Session 持有沙箱配置）
- contested_by: 无

### evidence
- supporting: EV-F001, EV-D001, SF-07
- contradicting: 无

### scope
- applies_when: 自举系统（Agent 管理沙箱/编译器编译自身/VM 运行在 VM 中）
- does_not_apply_when: 非自举系统（管理者和被管理者完全分离）

---

## KO-04 · 单一真相源与派生策略的一致性治理

| 字段 | 值 |
|------|-----|
| category | ARCHITECTURE / ENGINEERING / DESIGN |
| abstraction | L3 |
| value | B |
| epistemic_status | Validated Pattern |
| confidence | high |

### L1 工程事实
- PermissionProfile 是沙箱策略的单一真相源：file_system_sandbox_policy / network_sandbox_policy / sandbox_policy 均从 permission_profile 派生（session.rs:212-235）
- SessionConfiguration.permission_profile_state 三字段（constrained profile + active profile id + workspace roots）必须通过方法同步，注释明确要求"Keep ... in sync by using the methods below instead of mutating the fields independently"（session.rs:96-99）
- effective_permission_profile 支持环境级覆盖：优先使用环境配置，否则使用 session 级 materialize（session.rs:177-190）

### L2 工程知识
因为沙箱策略涉及多个维度（文件系统/网络/执行），如果各维度独立配置会导致策略冲突难以发现（如文件系统允许写入但网络禁止访问，或环境级与 session 级不一致）。所以用 PermissionProfile 作为单一真相源，所有派生策略通过纯函数生成，确保一致性和可审计性。

### L3 工程模式
Single Source of Truth（SSOT）模式。
- 同类对照：数据库 schema 派生 API 层（DB 是 SSOT，API 从 schema 生成）、Kubernetes desired state 派生实际状态（declarative config 是 SSOT）、React 单一数据源（state 是 SSOT，UI 从 state 派生）
- codex 实例：PermissionProfile → FileSystemSandboxPolicy + NetworkSandboxPolicy + SandboxPolicy

### L4 认知模型
在多维度约束系统中，单一真相源 + 显式派生函数是确保一致性的可扩展方法。派生函数必须是纯函数（相同输入→相同输出），禁止在派生层直接修改。

### L5 方法论
1. 系统设计时识别核心约束维度，建立单一真相源
2. 所有派生策略通过纯函数从真相源生成
3. 禁止在派生层直接修改（如 session.rs 中禁止独立修改三字段）
4. 支持层级覆盖时，覆盖优先级必须明确（环境级 > session 级）
5. 派生函数的输入输出必须可审计（可追溯到具体的 PermissionProfile 配置）

### provenance
- discovered_by: code-analyst（EV-C005）+ authority-analyst（EV-A004）
- supported_by: 无
- contested_by: 无

### evidence
- supporting: EV-C005, EV-A004, SF-03
- contradicting: 无

### scope
- applies_when: 多维度策略/配置系统（安全策略/权限系统/构建配置）
- does_not_apply_when: 单一维度、无派生关系的简单配置

---

## KO-05 · Fail Closed 作为 AI 系统的安全默认

| 字段 | 值 |
|------|-----|
| category | AGENT / PERMISSION / SECURITY |
| abstraction | L4 |
| value | A |
| epistemic_status | Principle（codex 项目内验证；跨项目验证 pending） |
| confidence | high |

### L1 工程事实
- Guardian：超时/执行失败/格式错误 → 默认拒绝（fail closed）（guardian/mod.rs:12-13）
- 沙箱拒绝后升级重试有严格前置条件：escalate_on_failure + unsandboxed_allowed + approval_policy 允许 + 非 strict_auto_review，任一不满足则保持拒绝（orchestrator.rs:317-438）
- 附件拥有的网络策略 + 升级权限请求 → 直接拒绝"attachment-owned network policy cannot be bypassed by sandbox escalation"（orchestrator.rs:149-159）
- 多 Agent V2 子 Agent 并发超限 → AgentLimitReached 拒绝（execution.rs:44-60）
- Guardian 熔断器：连续拒绝达到阈值 → InterruptTurn（guardian/mod.rs:196-232）

### L2 工程知识
因为 AI 系统的误放（错误地允许了危险操作）代价远高于误拒（错误地拒绝了安全操作）——误放可能导致未授权变更/数据泄露/系统破坏，而误拒只需用户手动确认。所以所有安全关键 Gate 统一采用 fail closed：不确定时默认拒绝。

### L3 工程模式
安全默认模式（Secure by Default / Fail Closed）。
- 同类对照：防火墙默认拒绝所有入站连接、权限系统默认禁止所有操作（需显式授权）、加密默认启用（HTTPS/TLS）、数据库默认不允许远程连接
- codex 实例：Guardian fail closed / 沙箱升级条件严格 / 附件网络策略不可绕过 / 多 Agent 超限拒绝 / 熔断器中断

### L4 认知模型
在 AI 代理系统中，fail closed 不是保守，而是对智能不确定性的结构性回应。AI 的判断本质上是概率性的，安全关键决策不能依赖概率性判断的"大概率正确"，必须在不确定时默认安全状态（拒绝）。

### L5 方法论
1. AI 系统的所有安全关键 Gate 必须 fail closed（不确定时默认拒绝）
2. fail closed 的拒绝必须可审计（记录拒绝原因、输入上下文、决策耗时）
3. fail closed 的拒绝必须可申诉（提供明确的升级路径，如人工审批）
4. 系统应提供明确的"白名单"机制（Never 审批策略），让用户可以显式选择自动放行低风险操作
5. 连续拒绝应触发熔断器，防止无限重试消耗资源或绕过安全 Gate

### provenance
- discovered_by: failure-analyst（EV-F004）+ authority-analyst（EV-A001, EV-A006）+ code-analyst（EV-C004, EV-C006）
- supported_by: 无
- contested_by: 无

### evidence
- supporting: EV-F004, EV-A001, EV-A006, EV-C004, EV-C006, SF-02, SF-06, SF-08
- contradicting: 无

### scope
- applies_when: 任何拥有自主执行能力的 AI Agent 系统的安全关键路径
- does_not_apply_when: 非安全关键路径（如 UI 渲染、日志记录）

---

## KO-06 · 大型代码库的核心模块反膨胀治理

| 字段 | 值 |
|------|-----|
| category | ENGINEERING / TOOLING / WORKFLOW |
| abstraction | L3 |
| value | B |
| epistemic_status | Validated Pattern |
| confidence | high |

### L1 工程事实
- codex-core 有 472 个源文件（core/src/ 递归统计），是最大的 crate
- AGENTS.md 明确指出"codex-core crate has become bloated because it is the largest crate, so it is often easier to add something new to codex-core rather than refactor out"（AGENTS.md:72-74）
- AGENTS.md 规定"resist adding code to codex-core"，引入新概念时应考虑是否有现有 crate 可放或应新建 crate，code review 时 push back 不必要的 core 添加（AGENTS.md:76-83）
- Session 结构体是 core 的核心状态容器，持有 20+ 字段（session.rs:42-78）

### L2 工程知识
因为"添加到 core 比重构出新 crate 更容易"导致路径依赖——开发者在时间压力下倾向于把新功能塞进 core，core 越大越难拆分，形成正反馈循环。所以需要显式治理规则（AGENTS.md 中的明确规定 + code review push back）来对抗膨胀。

### L3 工程模式
模块边界治理模式（Anti-corruption Layer / Module Boundary Enforcement）。
- 同类对照：微服务拆分（从 monolith 中提取服务）、monorepo package 边界（Nx/Turborepo 的依赖方向约束）、Linux 内核的子系统维护（各子系统有明确维护者和准入标准）
- codex 实例：AGENTS.md 作为治理文档，规定 core 的准入标准，code review 作为执行机制

### L4 认知模型
大型代码库中，核心模块的膨胀是结构性倾向（路径依赖 + 正反馈），必须通过显式治理规则 + code review 来对抗，而非依赖开发者自觉。

### L5 方法论
1. 核心模块应定义明确的准入标准（什么可以放进来，什么应该放外面）
2. 新增功能优先考虑现有非核心模块或新建模块
3. code review 应主动 push back 核心模块的非必要添加
4. 当核心模块超过一定规模（如文件数/行数阈值）时，应启动拆分计划
5. 治理规则必须文档化（如 AGENTS.md），不能仅靠口头约定

### provenance
- discovered_by: doc-analyst（EV-D002）+ code-analyst（EV-C001）
- supported_by: 无
- contested_by: 无

### evidence
- supporting: EV-D002, EV-C001, SF-04
- contradicting: 无

### scope
- applies_when: 大型代码库（>100 文件或 >10K 行的核心模块）
- does_not_apply_when: 小型项目（<5K 行，单模块即可管理）

---

## KO-07 · 产生→验证→授权→执行→记录的五段式架构指纹

| 字段 | 值 |
|------|-----|
| category | ARCHITECTURE / AGENT / DESIGN |
| abstraction | L4 |
| value | A |
| epistemic_status | Validated Pattern（codex 项目内 4 处独立验证；跨项目验证 pending） |
| confidence | high |

### L1 工程事实
在 codex 中观察到 4 处独立的"产生→验证→授权→执行→记录"五段式结构：
1. **工具执行**：模型产生 tool_call → Approval Gate 验证 → Sandbox 授权 → tool.run() 执行 → Event 记录（F-01, F-05）
2. **Guardian 审批**：ApprovalContext 产生 → transcript 验证 → Guardian 授权 → 执行放行 → 熔断器记录（F-04）
3. **网络访问**：请求产生 → begin_network_approval 验证 → proxy 授权 → 执行 → DeferredNetworkApproval 记录（EV-A005）
4. **Session 初始化**：配置产生 → 并行 setup 验证 → 权限配置授权 → Session 创建 → SessionConfigured Event 记录（F-02）

### L2 工程知识
因为每一段对应一个独立的关注点（意图/正确性/权限/动作/可追溯），分离后每段可独立审计、独立升级、独立失败。如果合并（如"产生即执行"），则无法在执行前验证和授权，也无法在执行后追溯。

### L3 工程模式
关注点分离的五段式执行链模式。
- 同类对照：网络协议栈的封装（应用层产生→表示层编码→会话层授权→传输层执行→链路层记录）、数据库的 WAL（操作产生→日志验证→锁授权→执行→WAL 记录）、Web 的 MVC（请求产生→路由验证→权限授权→Controller 执行→日志记录）
- codex 实例：4 处独立出现的五段式结构

### L4 认知模型
复杂代理系统的执行路径自然分解为产生→验证→授权→执行→记录五段，这是可审计代理系统的架构指纹。每一段的独立性是系统可审计性和可恢复性的基础。

### L5 方法论
1. 设计代理执行链时显式划分五段：产生（Intent）→验证（Validation）→授权（Authorization）→执行（Execution）→记录（Audit）
2. 每段有独立的失败模式和升级路径（验证失败→拒绝；授权失败→升级；执行失败→重试/降级）
3. 记录段必须不可篡改地捕获前四段的决策（谁产生/谁验证/谁授权/执行了什么）
4. 五段可以在实现上合并（如验证和授权在同一个函数），但逻辑上必须可区分
5. 系统的可审计性取决于记录段的完整性，而非执行段的正确性

### provenance
- discovered_by: pattern-miner（PAT-01）
- supported_by: code-analyst（EV-C003, EV-C002）+ authority-analyst（EV-A001, EV-A002, EV-A005）
- contested_by: 无

### evidence
- supporting: PAT-01, SF-01, SF-02, SF-08, F-01, F-02, F-04, F-05
- contradicting: 无

### scope
- applies_when: 任何需要可审计性的代理/自动化系统
- does_not_apply_when: 一次性脚本（无需审计和追溯）

---

## Knowledge Object 统计

| ID | 标题 | 层级 | 价值 | 认知状态 | 证据数 |
|----|------|------|------|---------|--------|
| KO-01 | Intelligence ≠ Authority | L4 | A | Validated Pattern | 5 |
| KO-02 | 声明 ≠ 证据 | L4 | A | Validated Pattern | 4 |
| KO-03 | 自举系统的递归矛盾与逃逸阀 | L3 | B | Validated Pattern | 3 |
| KO-04 | 单一真相源与派生策略 | L3 | B | Validated Pattern | 2 |
| KO-05 | Fail Closed 安全默认 | L4 | A | Principle | 5 |
| KO-06 | 核心模块反膨胀治理 | L3 | B | Validated Pattern | 2 |
| KO-07 | 五段式架构指纹 | L4 | A | Validated Pattern | 4 |
| **合计** | | | **4A + 3B** | | **25** |

### Most Important Finding
**KO-01（Intelligence ≠ Authority）+ KO-05（Fail Closed）+ KO-07（五段式架构指纹）共同构成了 codex 作为 Agent 系统的核心设计哲学：AI 的判断力与执行权必须分离，执行权由多 Gate 链控制，所有安全关键路径 fail closed，执行链遵循产生→验证→授权→执行→记录的五段式结构。** 这是 codex 项目最可迁移的架构认知。
