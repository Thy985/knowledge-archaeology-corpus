# 02 — Engineering Knowledge（EK Graph）

> 规则：每条 EK 声明 `links`（mechanism/subsystem/causal/dependency/constraint/contrast）。全部证据可回溯 `aigis/`、`ARCHITECTURE.md`、`tests/`。

## EK-01 — 确定性检测管线 L1-L3：模式→语义→解码 三级递进
**Fact**：L1 正则（165+ patterns / 25+ 类别 × 4 语言，NFKC 归一化→zero-width 移除→空格压缩→Confusable→Emoji 移除；per-category score cap = base×2）→ L2 语义相似度（56 攻击短语词典，**只查 L1 未命中类别**防双重检测，difflib.SequenceMatcher + n-gram）→ L3 条件解码（仅编码指示时激活：base64/hex/ROT13/URL/Unicode；解码后重扫 L1/L2，按 rule_id 去重，"(decoded)" 标注）（ARCHITECTURE Detection Pipeline）。
**Why**：三层成本递增、覆盖面互补；每层只补上层缺口，避免重复计分与性能浪费。
**links**：`causal` EK-01→EK-03（解码产物重新进入判定）；`constraint` EK-01 约束 EK-05（scorer 汇总各层 MatchedRule）。

## EK-02 — MatchedRule 可解释决策单元（rule_id/score_delta/owasp_ref）
**Fact**：检测命中产出 MatchedRule（rule_id, score_delta, owasp_ref）；risk_score 由规则累积；matched_rules 随 ActivityEvent 进入治理流（ARCHITECTURE step 4）。
**Why**：决策必须可解释、可追溯（哪个规则、加多少分、对应 OWASP 哪条）——安全工具的审计要求。
**links**：`mechanism` EK-02↔EK-06（审计日志记录同源规则）；`dependency` EK-02 依赖 EK-01（规则来源）。

## EK-03 — Active Decoding 按需激活（性能纪律）
**Fact**：L3 仅在检测到编码指示（base64 特征/hex/ROT13 指示/URL/Unicode 转义）时执行解码重扫——"minimizing performance impact"（ARCHITECTURE L3 注释）。
**Why**：全量解码成本高且误报多；特征指示门控把重活限制在可疑输入。
**links**：`contrast` EK-03↔EK-01（条件执行 vs 全量执行）；`mechanism` EK-03↔EK-04（都是"只做必要的事"）。

## EK-04 — L4 CaMeL：Capability Tokens + Taint 追踪（权限边界工程化）
**Fact**：taint.py 给外部输入（user/RAG/MCP）打 taint 标签并追踪传播；tokens.py 按能力发 token（file:read/net:connect/exec:shell）；enforcer.py taint 级 × 所需权限 → allow/deny，自动阻止被污染数据的高权操作（aigis/safety/ + ARCHITECTURE L4，引用 CaMeL 论文 2025）。
**Why**：注入攻击的根因是"不可信数据被信任执行"；把数据可信度（taint）与操作权限（capability）正交分解，是结构化防御。
**links**：`constraint` EK-04 约束 EK-09（FSM 状态迁移也需权限意识）；`dependency` EK-04 依赖 EK-01（taint 标签由检测结果初始化）。

## EK-05 — L5 Atomic Execution Pipeline：Scan-Execute-Vaporize
**Fact**：AEP 三段——Scan（预执行 L1-L4 检查命令/参数）→ Execute（sandbox.py 受限文件系统/网络隔离执行）→ Vaporize（vaporizer.py 安全擦除临时文件与内存敏感数据）（ARCHITECTURE L5 + aigis/safety/）。
**Why**：危险操作不死禁而是隔离执行 + 痕迹清零——"执行后不可追查"与"执行中不可越权"双保险。
**links**：`mechanism` EK-05↔EK-10（都是"有限信任"族）；`causal` EK-05→EK-07（执行结果要验证）。

## EK-06 — 审计日志：HMAC-SHA256 + Hash Chain + Frozen Entry（证据不可抵赖）
**Fact**：SignedAuditLog entry 为 dataclass frozen（不可变）；签名 = HMAC-SHA256(canonical JSON sort_keys)；每条含 prev_hash 形成链；verify_chain 返回断裂点序号；首条引用 genesis hash（audit/signed_log.py + chain.py）。
**Why**：安全日志必须防篡改——改任意字段 → 签名失效 + 链断裂；frozen 从语言层杜绝意外突变。
**links**：`mechanism` EK-06↔EK-02（可解释规则进入不可抵赖日志）；`constraint` EK-06 约束 EK-14（alerts 永久保留才有链意义）。

## EK-07 — L6 声明式安全规约（no_exfil/no_exec/pii_guard + 自定义 YAML/JSON）
**Fact**：safety spec（spec.py/builtin_specs.py）内建 no_exfil（禁外泄）/no_exec（禁任意执行）/pii_guard（防 PII 泄漏）；loader 支持 YAML/JSON 自定义 spec；verifier 校验执行结果是否满足规约（ARCHITECTURE L6 + aigis/spec_lang/）。
**Why**：把安全属性声明化（而非散落代码），可静态审查、可组合、可对执行结果做形式验证。
**links**：`dependency` EK-07 依赖 EK-05（验证对象是 AEP 执行产物）；`constraint` EK-07 约束 EK-09（FSM 迁移需满足 spec）。

## EK-08 — L7 Goal-Conditioned FSM：AgentStateMachine + FSMMonitor
**Fact**：spec_lang/fsm.py：StateRule（状态规则）、AgentStateMachine、FSMViolation、FSMMonitor——监控 Agent 状态迁移是否违反目标约束；被定位为 alignment-frontier 10 个 miss（sandbox escape 等）的验证载体（README benchmark 自述）。
**Why**：目标漂移/子 Agent 共谋等运行时行为问题需要状态级监控，而非单次调用判定。
**links**：`mechanism` EK-08↔EK-04（运行时行为治理族）；`causal` EK-08→EK-12（违规进入 incident）。

## EK-09 — PolicyDSL：Trigger/Predicate/Enforcement 三段式规则
**Fact**：spec_lang/parser.py：Trigger（触发条件）、Predicate（谓词评估）、Enforcement（执行动作）、Rule、PolicyDSL；evaluator.py：RuleEvaluator + _tool_matches（模式匹配工具名）+ _get_actual_value（按 pred_type 取值）；决策留档：PENDING_DECISIONS D7 选 YAML-based（"consistent with existing policy.yaml, zero learning curve"）。
**Why**：策略即配置（非代码），审计者/合规官可读；DSL 语法选择有显式决策记录。
**links**：`mechanism` EK-09↔EK-07（声明式族）；`dependency` EK-09 依赖 EK-01（predicate 需要检测结果作为上下文）。

## EK-10 — 零核心依赖哲学（可审计供应链）
**Fact**：pyproject `dependencies = []`；可选依赖（pyyaml/fastapi/cryptography）显式隔离；PENDING_DECISIONS D8 审计签名选 HMAC-SHA256（stdlib）而非 Ed25519（"stdlib only, simple key management"），D9 跨会话存储选 JSON（"zero dependencies, consistent with project philosophy"）。
**Why**：安全工具的依赖面 = 攻击面（供应链投毒直击信任链）；stdlib 优先是刻意取舍。
**links**：`constraint` EK-10 约束 EK-06（HMAC 而非 Ed25519 正是此哲学的产物）；`contrast` EK-10↔EK-07（可选依赖 vs 核心零依赖边界）。

## EK-11 — MCP 3-stage Scanner + Trust Score（供应链信任评估）
**Fact**：mcp_scanner.py：scan_mcp_tool（definition+invocation+response 三阶段）+ analyze_permissions（file_system/network/code_execution/sensitive_data 四维）+ detect_rug_pull（快照对比）+ score_server_trust（100 - avg_risk - permission_pen；70-100 trusted / 40-69 suspicious / 0-39 dangerous）；ARCHITECTURE 列出 MCP 6 attack surfaces（description 注入/schema 隐藏描述/output re-injection/cross-tool shadow/rug pull/sampling hijack）。
**Why**：MCP 是 Agent 能力扩展主通道，工具定义即攻击面；快照对比可检测"恶意变更"（rug-pull）。
**links**：`mechanism` EK-11↔EK-06（快照也是证据，需防篡改配合）；`causal` EK-11→EK-12（危险 MCP → incident）。

## EK-12 — Incident Response：NIST SP 800-61 四阶段 + 2 层设计 + SLA
**Fact**：Detection（scan→triage→route→notify）→ Containment（auto-block→Incident created→timeline→SLA）→ Eradication/Recovery（review approve→replay→output filter→mitigated）→ Post-Incident（weekly report→recommendations→auto-fix suggest→closed）；enterprise_mode=true 启用（默认零配置）；incidents 表（PostgreSQL，25+ 字段 + JSONB timeline + request snapshot 供 replay，INC-YYYY-NNNN 编号）（ARCHITECTURE Incident v0.0.3）。
**Why**：安全事件要有生命周期管理（而非一次判定）；SLA 时钟让处置可度量；replay 支持事后复盘。
**links**：`causal` EK-12→EK-13（事件进周报闭环）；`dependency` EK-12 依赖 EK-06（事件证据来自审计日志）。

## EK-13 — 对抗反馈循环：adversarial_loop + auto_fix（自我改进机制）
**Fact**：aigis/adversarial_loop.py + auto_fix.py + weekly_report.py；README 自述 "Weekly adversarial loop auto-generates new detectors from observed bypasses. v1.0.0→v1.1.0 was 21 patch releases and ≈60 new detectors in 8 days"。
**Why**：检测器是"对抗博弈"产物——观察绕过 → 生成新规则 → 回归，是持续改进而非一次性交付。
**links**：`mechanism` EK-13↔EK-01（新规则进 L1 patterns 族）；`causal` EK-13←EK-12（事件复盘喂给对抗循环）。

## EK-14 — 3 层日志 + alerts 永久保留
**Fact**：project 级 `.aigis/logs/*.jsonl`（7 天压缩、60 天删除）+ global `~/.aigis/global/` + **alerts `~/.aigis/alerts/`（仅 block/review，永久保留）**（ARCHITECTURE Log 3-Tiers）。
**Why**：日常日志可裁剪，但安全告警是不可丢证据——保留策略分级。
**links**：`dependency` EK-14 依赖 EK-06（告警也要签名链）；`contrast` EK-14↔EK-06（保留策略 vs 防篡改机制）。

## EK-15 — 能力边界诚实声明（93.5% / 0.0% FP / 10 miss 不掩饰）
**Fact**：README benchmark 表：整体 93.5%（144/154）+ 0.0% FP（0/26）；10 个 miss 明确列出（sandbox escape/self-privilege escalation/audit tampering/evaluation gaming/CoT deception）并标注 "tracked on the L6/L7 verifier roadmap, not claimed as solved"。
**Why**：安全工具的诚实边界 = 可用性声明——不虚报检出率；miss 类别进路线图形成验证循环。
**links**：`constraint` EK-15 约束 EK-08（miss 类别正是 L7 FSM 的待验证目标）；`contrast` EK-15↔EK-13（诚实边界 vs 改进循环的张力）。

## EK-16 — i18n 语言检测的环境耦合缺口（测试环境发现）
**Fact**：detect_lang 优先级 arg > AIGIS_LANG > LC_ALL > LANG > 默认 en（i18n.py:154-166，符合 POSIX LC_ALL>LANG 标准）；test_unknown_env_falls_through 只 mock AIGIS_LANG/LANG 未清 LC_ALL——沙箱全局 LC_ALL=en_US.UTF-8 触发失败（实测返回 en 而非期望 ko）。
**Why**：测试对环境的隐式假设（无 LC_ALL）在 CI/沙箱不一致时产生假失败；实现本身符合标准，缺陷在测试隔离。
**links**：`contrast` EK-16↔EK-10（零依赖哲学 vs 环境依赖测试的张力）；`constraint` EK-16 约束 EK-01（i18n 影响多语言 pattern 选择）。

## EK Graph 质量自检
- 16 条 EK 全部有 links（平均出边 >1）；孤立 0；聚合覆盖率 100%（见 03）。

## EK-17 — 异常语义分层：判定 fail-closed / 日志 fail-open（Auditor Reconciliation 增量）
**Fact**：claude_code.py 适配器——scan 异常 → 打印 "blocking (fail-closed)" + sys.exit(2)；policy 异常 → decision="deny"；仅 ActivityStream 记录异常 → pass（判定不受影响）。activity.py __post_init__：user_id 获取失败 → "unknown"（无害 fallback）。
**Why**：安全工具的关键路径异常默认拒绝（宁可误伤不可放行），审计缺口单独容忍（可用性取舍）——异常行为也分层。
**links**：`mechanism` EK-17↔EK-06（审计与判定分离）；`constraint` EK-17 约束 EK-12（incident 流程依赖判定可靠）。

## EK-18 — 用户扩展面加固：_regex_guard 安全编译（Auditor Reconciliation 增量）
**Fact**：_regex_guard.py 对用户自定义 custom_rules 的 regex 做安全编译——拒绝 re.error 崩溃 / 拒绝吞异常静默禁用 / 拒绝 ReDoS 灾难回溯；safe_compile_user_regex 返回 None 视为 hard failure（log + matched-rule marker），禁止静默继续。
**Why**：安全工具允许用户扩展规则 = 把"用户代码"引入信任链；扩展面本身必须被加固（fail 可见而非静默失效）。
**links**：`mechanism` EK-18↔EK-10（扩展面与依赖面同权治理）；`constraint` EK-18 约束 EK-01（用户规则进入 L1 前先过编译门）。
