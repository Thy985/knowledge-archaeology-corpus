# Job Manifest — ARCH-2026-09-26-001

## run 信息
- **run_id**: ARCH-2026-09-26-001
- **日期**: 2026-09-26（每日 Knowledge Archaeology Runtime SOP 定时触发）
- **skill_version**: knowledge-archaeology v3.2
- **仓库基线**: KnowlegeMap HEAD `457ea7d`（09-26 雷达，第 24 次扫描）；corpus main HEAD `aef17c4`

## 候选排序（按 Knowledge Value / Agent-AI Engineering 相关性 / Novelty / 增长信号 / 近期发布 / Corpus 连接 / 是否已考古 / Benchmark 学习价值，不按 star）

| # | 候选 | 等级 | 状态判定 | 考古性 | 核心理由 |
|---|---|---|---|---|---|
| 1 | **OWASP Agentic Skills Top 10** | S | QUEUED→**SELECTED** | ✅ 可考古（`OWASP/www-project-agentic-skills-top-10`，241★，2026-08-12 活跃） | 昨日排序第 2 的 OWASP S 卡唯一可直接考古对象；与 MCP（09-25 考古，SEP-2640 嵌套技能同意门）直接闭环、与已归档 skillfortify（skill 供应链治理）/agent-governance-toolkit（AGT 映射 ASI01-10）三重 Corpus 连接；Skills Top10 v1.0 目标 2026 Q3 = 近期发布信号；Benchmark 学习价值极高（威胁建模 ↔ 协议实现 ↔ 供应链治理三向对照） |
| 2 | agentic-attack 范式事件 | S | QUEUED（重大事件日 5） | ❌ 事件集合无单一可考古仓库（Unit42/GTIG/OpenAI/Gambit 均为报告） | 今日 Medicare 政府级首例 + $25/目标成本崩塌 + Muse 个人端点零日，威胁范式冲击最大；但无代码/规范仓库可浅克隆 → 维持候选观察，增量已入库 |
| 3 | Agent Harness / Control Plane | S | QUEUED | ⚠️ 部分已考古（omnigent 09-07 已归档）；MS Agent Framework 非开源仓库 | 六职责综述与自研验证编译器同构；JIT-Agent（09-03）新增 → 维持观察 |
| 4 | Agent Memory 2026 | S | QUEUED | ⚠️ 核心框架已考古（mem0/letta-code/dsh-memory-evolve 均在 corpus）；Zep 厂商自报待独立复测 | 记忆厂商数字大战需证据纪律甄别 → 维持观察 |
| 5 | PI-Hunter 注入审计 | A | QUEUED | 待探 | 注入攻击面工具化 |
| 6 | Agentic CLEAR | A | QUEUED | 待探 | 评测/信任 |
| 7 | computer-browser-use | A | QUEUED | 待探 | Claude for Chrome 增量（09-24） |
| 8 | agent-formal-verification | A | QUEUED | 待探 | BenchShield + 方法论翻转（09-26 增量） |

## 选中项目
- **project**: OWASP Agentic Skills Top 10（AST01-AST10 + Trust Boundary Model B1-B4）
- **repository**: https://github.com/OWASP/www-project-agentic-skills-top-10.git
- **mode**: initial
- **priority**: S
- **reason**: ① S 级卡唯一未考古且可直接考古对象；② 与 09-25 MCP 考古（Skills over MCP / SEP-2640 嵌套技能 fresh consent / allowed-tools 审批门）形成"skill 供应链威胁模型 ↔ 协议实现"闭环，直接验证 MCP 轮 EK-25/C-08；③ 与已归档 skillfortify（skill 供应链安全）、agent-governance-toolkit（AGT 安全治理）同域连接；④ Skills Top10 v1.0 目标 2026 Q3 + Trust Boundary Model 新增 = 近期发布/增长信号；⑤ Benchmark 学习价值：威胁分类法（AST01-10）可作为 knowledge-archaeology 自身评估"技能安全"的知识底座
- **source_entry**: `03_expansion_queue/candidates/[cand]owasp-agentic-security.md`（OWASP 体系卡，本考古聚焦其 Skills Top10 子域）

## 考古边界
- 只读 OWASP Skills Top10 仓库（含 AST 威胁分类、Trust Boundary Model、mapping 到 ASI）
- 不修改 KnowlegeMap / 生产 Skill / 目标仓库
- 跨协议假说验证：MCP 轮 C-08（扩展默认关闭 → 事实分裂）与 OWASP "Update Drift / Weak Isolation" 对照；skillfortify 轮 skill 供应链治理 ↔ OWASP Skills Top10
