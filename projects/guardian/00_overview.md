# 00 — Overview：Guardian（legionforge-guardian，确定性 Agent 安全 Sidecar）

## 一句话定位

Guardian 是 **FastAPI sidecar（:9766）**：在任意 Agent 框架执行工具调用前跑 **7 项确定性检查**（JWT Task token ACL → 工具注册表 → 能力边界 → 破坏性模式 → 序列契约 → 哈希完整性 → 自适应规则），fail-fast 顺序执行，**无 LLM、无启发式**——"decisions that cannot be prompt-injected"；审计走 PostgreSQL SHA-256 hash chain，内置 canary 蜜罐工具。

## 为什么选它（Job Selection 摘要）

- 雷达 #14（09-14）"下一步"第一优先级 = **Aigis/Guardian/AI Protector 三选一实测**（防火墙品类）；Aigis 已于 09-13 考古完成，本日选**第二个数据点 Guardian**。
- **机制对照价值最高**：Guardian 与 Aigis 同属"确定性防火墙"品类但实现路径不同（Task token ACL vs CaMeL capability tokens；sidecar 独立进程 vs middleware 嵌入）——直接支撑 KO-01/KO-03 从单案例走向双项目互证。
- 规模 10 .py 可 **100% 覆盖**（app.py 1412 行承载全部核心），考古完整度最高。
- fresh：2026-09-10 发布；"No LLM. No heuristics." 是品类内最强反注入宣称。

## 三个核心发现（Top Findings）

### F1 — "decisions that cannot be prompt-injected"：把判定从文本空间移到结构空间
Guardian 的关键洞察：LLM 检查器本身可被注入，所以判定输入不是"自然语言"而是**结构化工具调用**（tool_id + args + action + task_token）。7 项检查全部操作结构化字段（JWT scope / 注册表 / 能力名单 / regex / 序列前缀 / 哈希 / 规则表），无任何自由文本语义判定 → 注入面被结构性消除。与 Aigis 的"输入文本检测管线"形成品类内两种哲学。

### F2 — 响应分级（halt / sandbox / log-only）而非二元 allow/deny
check 3（破坏性模式）区分 FORCE_END_CATEGORIES（halt）与 HITL_LOG 类（允许 + 后台记录 threat_events）；check 4（序列契约）对"novel sequence"用 **sandbox** 而非 halt——未知行为被隔离而非阻断。三档响应让"防御强度 × 可用性"可调，这是 Aigis 二元决策没有的粒度。

### F3 — 工程治理完备度远超规模（6 CI workflow + ZAP DAST + 审计 hash chain + canary）
256K 的小项目带：6 个 GitHub Actions（含 **dast.yml = OWASP ZAP 主动扫描**、oss-audit、zizmor）、Action 全部固定 SHA、gitleaks、dependabot+renovate、PostgreSQL 审计 hash chain（prev_hash/genesis）、**canary 蜜罐工具**（guardian_canary：被调用即探测证据）——"安全工具自身的安全工程"是品类共识的第二数据点（与 Aigis 的 SECURITY.md/零依赖同哲学）。

## 质量指标速览

- EK：14 条；KO：4 个；Candidates：4 个
- 测试证据：**57 passed / 9 skipped（1.44s）**——单元测试全绿（9 skipped 为 live 测试需真实 sidecar 环境）
- 独立验证：ACCEPT（详见 06）

## 已知边界（诚实声明）

- **pyproject v0.1.1 vs README 服务 version "4.0.0" 不一致**（README 超前或语义版本漂移，见 C-04）。
- 未跑 live 测试（需 GUARDIAN_TEST_URL 真实服务 + PostgreSQL）——hash chain 数据库端行为以代码+init.sql 为准（S3），非 S5 实测。
- GitHub API 限流未取 stars/created/pushed 元数据（core 0/60）；仓库存在性与 HEAD 经 git ls-remote 确认。
- 9 个 skipped 中 8 个为 live 测试（环境依赖），非产品失败。
