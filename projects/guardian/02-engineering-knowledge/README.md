# 02 — Engineering Knowledge（EK Graph）

> 规则：每条 EK 声明 `links`（mechanism/subsystem/causal/dependency/constraint/contrast）。全部证据可回溯 `src/legionforge_guardian/app.py`、`init.sql`、`tests/`、`.github/workflows/`。

## EK-01 — 7 项确定性检查 fail-fast 编排（结构化判定管线）
**Fact**：/check 端点按序执行 Check 0-6（Task token ACL→registry→capability→destructive→sequence→hash→adaptive），首个失败立即 halt/sandbox；"Seven checks in order, fail-fast. NO LLM calls."（app.py:1194-1202）。
**Why**：判定必须确定性且快速（"hot path"）；把注入面从自然语言移到结构化字段（tool_id/args/action/token），无自由文本语义判定。
**links**：`mechanism` EK-01↔Aigis-EK-01（Aigis L1-L7 管线同族：确定性判定，跨项目对照）；`causal` EK-01→EK-03（registry 是后续检查前提）。

## EK-02 — Task token ACL：JWT scope 授权（检查 0）
**Fact**：_validate_task_token 验 JWT 签名/过期；granted_tools 集合外的工具 → halt（TOOL_SCOPE_VIOLATION）；**无 token 请求跳过检查**（"unconstrained for backward compatibility, Phase 4 will enforce"）（app.py:611-651）。
**Why**：任务级最小权限（一个任务 token 只授权该任务需要的工具）；向后兼容窗口是显式设计决策（渐进强制）。
**Reconciliation 补充（Auditor）**：token 授权维度实际为**三级资源粒度**——granted_tools（工具）+ granted_tables（数据表）+ granted_data_classes（数据类），且 escalation_policy 默认 "deny"（升级策略默认拒绝）（app.py:276-290 + 329-341）——任务最小权限覆盖工具/表/数据类三个层面。
**links**：`constraint` EK-02 约束 EK-01（ACL 是首个检查，先于一切）；`contrast` EK-02↔EK-04（scope 白名单 vs 能力黑名单）。

## EK-03 — Tool registry + revocation 优先（检查 1）
**Fact**：revoked 工具先于 approved 检查（"Phase 6: revocation is checked FIRST"）；注册表缓存 _CACHE_TTL_SECONDS=10s 传播；未注册 → halt（CAPABILITY_VIOLATION）（app.py:653-680 + init.sql tool_registry）。
**Why**：撤销必须立即生效（优先于批准）；10s 传播是缓存一致性取舍。
**Reconciliation 补充（Auditor）**：DB 查询失败 → **fallback 到 in-process 注册表（stale data）而非阻断**（"Deliberately broad: any DB/query error must fall back to stale data, not crash the caller"，app.py:565-573）——撤销传播存在已知边界窗口（DB 不可达时已撤销工具可能短暂放行），是 fail-closed 热路径的显式例外。
**links**：`dependency` EK-03 依赖 EK-07（注册时记录 hash，检查时比对）；`mechanism` EK-03↔EK-05（注册表与 canary 都在注册层）。

## EK-04 — Capability boundary 双路径修复（检查 2 + Gap 2 fix）
**Fact**：FORBIDDEN_CAPABILITIES（register_tool/write_executable/spawn_agent_direct/escalate_scope 等 8 项）双检查：action 类型 + tool_id 名称；注释 "Gap 2 fix: was unreachable because action was always 'invoke'"（app.py:681-712 + 115-125）。
**Why**：曾经 action 恒为 "invoke" 使 tool_id 路径不可达——修复史证明"黑名单需要双入口"（类型与名称都可绕过）。
**links**：`contrast` EK-04↔EK-02（黑名单 vs 白名单互补）；`causal` EK-04←EK-01（fail-fast 暴露此缺口）。

## EK-05 — 破坏性模式 9 regex 族 + 分级响应（检查 3）
**Fact**：_GUARDIAN_DESTRUCTIVE_PATTERNS 9 族（credential probing/shell injection/bulk exfiltration/data staging/recon/privilege escalation/self-probe 等）；FORCE_END_CATEGORIES（CMD_INJECTION/SELF_PROBE…）→ halt，其余 → **log-only（允许 + 后台 threat_events）**（app.py:127-135 + 853-882 + 255-264）。
**Why**：分级响应（halt/log-only）让防御强度可调——证据收集类威胁记录放行，即时破坏类阻断；parity 测试强制与 src.security.core 同步（test_guardian_destructive_patterns_count_matches_core）。
**links**：`mechanism` EK-05↔EK-02（分级 vs 二元）；`constraint` EK-05 约束 EK-01（regex 族是检查 3 的规则源）。

## EK-06 — Sequence contracts：playbook 前缀匹配（检查 4）
**Fact**：agent_profiles 注册 sequence（TEXT[]）；候选序列必须是某注册序列前缀，novel sequence → **sandbox**（SEQUENCE_VIOLATION）；无注册序列 agent 不受限（渐进 rollout）（app.py:885-917 + init.sql agent_profiles）。
**Why**：工具调用的"行为轨迹"是合法性的另一维度——单次合法调用的组合可能是攻击链；前缀匹配是简单可验证的契约。
**links**：`mechanism` EK-06↔Aigis-EK-08（Aigis L7 FSMMonitor 同族：行为轨迹治理，跨项目对照）；`contrast` EK-06↔EK-05（序列级 vs 单次级）。

## EK-07 — Hash integrity：schema/description 篡改检测（检查 5）
**Fact**：注册时记录 description_hash/schema_hash（verify_tool_before_invocation）；检查时比对进程内注册表哈希，不匹配 → halt（TOOL_HASH_MISMATCH）（app.py:918-947 + init.sql tool_registry hash 列）。
**Why**：工具定义（描述/schema）是注入载体——定义被改 = 能力被替换；哈希基线是最轻量的篡改检测。
**links**：`mechanism` EK-07↔Aigis-EK-11（Aigis MCP rug-pull 快照同族：定义基线对比，跨项目对照）；`dependency` EK-07 依赖 EK-03（注册表持有基线）。

## EK-08 — Adaptive rules：DB 热加载 + 坏规则不崩热路径（检查 6）
**Fact**：threat_rules 表 60s 刷新；CAPABILITY_BLOCK/INJECTION_PATTERN/SEQUENCE_BLOCK 三类；**malformed regex → skip + warning（"bad rules must not break the hot path"）**（app.py:948-1034 + init.sql threat_rules）。
**Why**：规则可运行时治理（block/log/sandbox 无需重部署）；规则错误不能拖垮判定（fail-safe 在规则层）。
**links**：`mechanism` EK-08↔Aigis-EK-09（Aigis PolicyDSL 同族：策略可配置化，跨项目对照）；`constraint` EK-08 约束 EK-05（自适应注入 regex 叠加于静态 9 族）。

## EK-09 — SHA-256 hash chain 审计（audit_log + 写失败非致命）
**Fact**：audit_log 表（seq/ts/event_type/agent_id/payload/prev_hash/row_hash）；_compute_audit_row_hash_direct 对 canonical（seq|ts|event_type|agent_id|json sort_keys|prev_hash）做 SHA-256；首条引用 _AUDIT_LOG_GENESIS；INSERT 后 UPDATE row_hash；**写失败 → warning + return -1（"must never block the action it's auditing"）**（app.py:768-851 + init.sql audit_log）。
**Why**：审计防篡改（改任意字段 → 链断裂）+ 审计可用性优先（审计缺失不阻断业务）。
**links**：`mechanism` EK-09↔Aigis-EK-06（Aigis HMAC-SHA256 hash chain 同构——**跨项目第二数据点**）；`constraint` EK-09 约束 EK-01（所有判定结果都应进审计链）。

## EK-10 — Canary 蜜罐工具（guardian_canary）
**Fact**：init.sql seed 注册 guardian_canary（APPROVED + canary hash）；任何 /check 调用它 → CANARY_TRIGGERED threat event + halt（"immediate evidence of a probing attack or hallucinating model"）（init.sql 底部 + app.py:1250-1280）。
**Why**：蜜罐/金丝雀是"预期内永不触发的工具"——触发即告警信号，零误报（只对真实探测或幻觉响应）。
**links**：`mechanism` EK-10↔EK-08（都是运行时探测信号）；`dependency` EK-10 依赖 EK-03（能过 registry 检查才有意义）。

## EK-11 — hot path fail-safe：misconfig/auth 失败 → halt（绝不 fail-open）
**Fact**：/check 前序——GUARDIAN_REQUIRE_AUTH 时 misconfigured（TASK_TOKEN_SECRET 未设）→ halt（GUARDIAN_MISCONFIGURED）；Bearer 缺失/无效 → halt（GUARDIAN_AUTH_FAILURE）；注释 "never silently allow on the security hot path" / "never fail-open"（app.py:1199-1220）。
**Why**：安全热路径的默认态是拒绝——配置错误必须显式可见而非静默放行。
**links**：`mechanism` EK-11↔Aigis-EK-17（Aigis claude_code 适配器 scan/policy 异常 → exit(2) 同族：**跨项目 fail-closed**）；`contrast` EK-11↔EK-02（auth 层 fail-closed vs token 缺失 fail-open 的边界）。

## EK-12 — 工程治理完备度（6 CI workflow + ZAP DAST + Action SHA 固定）
**Fact**：.github/workflows/ 6 个（ci/dast/lint-workflows/oss-audit/publish/test）；dast.yml 跑 OWASP ZAP 对 :9766 主动扫描（zap.yaml failOnError: true）；SECURITY.md 要求 Action 全部固定 commit SHA（无可变 tag）；gitleaks + dependabot + renovate + zizmor。
**Why**：安全工具自身的安全工程是可信度前提——供应链（Action 固定/依赖自动化）与应用（DAST/secret 扫描）双线治理。
**links**：`mechanism` EK-12↔Aigis-GOVERNANCE（Aigis SECURITY.md 72h 响应同族：治理闭环，跨项目对照）；`dependency` EK-12 依赖 EK-09（审计链是治理的证据底座）。

## EK-13 — sidecar 部署形态（独立进程 vs 嵌入）
**Fact**：Guardian 是独立 FastAPI 服务（:9766 + 自有 PostgreSQL）；init.sql 支持两种部署（standalone schema / 伴生 LegionForge DB 时不跑）；SDK 客户端通过 HTTP 调用（tests/test_sdk.py 用 respx mock）。
**Why**：sidecar 形态与框架解耦（"drop it in front of any agent framework"）——语言/框架无关的安全边界；代价是每调用一次网络往返。
**links**：`contrast` EK-13↔Aigis-middleware（Aigis middleware 嵌入形态——**部署哲学对照**）；`constraint` EK-13 约束 EK-11（网络层 auth 是 sidecar 前提）。

## EK-14 — 版本号不一致（pyproject 0.1.1 vs README "4.0.0"）
**Fact**：pyproject version=0.1.1（Alpha）；README quickstart 显示健康检查返回 `"version": "4.0.0"`；CHANGELOG.md 存在但未在本轮核验版本史（浅克隆）。
**Why**：语义版本漂移或 README 超前——安全工具的版本可信度问题（用户依赖版本号判断补丁级别）。
**links**：`constraint` EK-14 约束 EK-12（治理完备 vs 版本管理不一致的张力）；`contrast` EK-14↔EK-01（工程严谨 vs 文档漂移）。

## EK Graph 质量自检
- 14 条 EK 全部有 links（平均出边 >1）；孤立 0；聚合覆盖率 100%（见 03）。
