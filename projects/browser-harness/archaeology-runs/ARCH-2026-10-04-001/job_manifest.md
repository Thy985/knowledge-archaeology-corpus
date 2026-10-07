# Job Manifest — ARCH-2026-10-04-001

## 选中项目
| 字段 | 值 |
|---|---|
| job_id | ARCH-2026-10-04-001 |
| project | browser-harness |
| repository | https://github.com/browser-use/browser-harness |
| mode | initial |
| priority | A（高） |
| source_entry | KnowlegeMap `03_expansion_queue/candidates/[cand]browser-harness-2026.md`（雷达 2026-09-02 捕获，A 级） |

## 候选排序（按 SOP 维度，非 star 排名）

| 排序 | 候选 | 状态 | 主要依据 |
|---|---|---|---|
| 1 | **browser-harness**（browser-use/browser-harness） | **NEW** | A 级卡；未考古；"agent 自愈写代码 harness"范式（CDP 裸连 + agent-workspace 内自生成工具）；与已考古 browser-use 主库（09-22）同厂延伸，构成天然对照；连接 E2E-CLI/campus_order/OpenClaw；15.7k stars、PyPI 0.1.6 活跃；中等规模适合单日深度考古 |
| 2 | A2A v1.0 refresh | REFRESH_REQUIRED | S 级卡；10-04 雷达重大增量（AAIF 托管 + v1.0 Agent Card 重构 + List Tasks 分页）；但 09-24 刚考古 10 天，且协议规范仓库工程密度低——下轮 refresh 候选 |
| 3 | MCP refresh（OpenAI Events + SEP-2577 + listen） | REFRESH_REQUIRED | S 级卡；10-04 雷达重大协议变化（stateless 化瘦身）；09-25 刚考古，同理顺延 |
| 4 | OpenCodeReview（阿里） | QUEUED | B 级卡；21K stars 但 pullfrog（10-02）同域刚考古，边际价值低 |
| 5 | agentic-inference-infra（Dynamo 等） | QUEUED | A 级卡；NVIDIA Dynamo 已考古（09-20）；其余为论文/规范线，无未覆盖主仓库 |
| 6 | agentic-attack-2026 | QUEUED | S 级卡但为事件/论文线，无单一可克隆仓库 |
| 7 | agentic-graphrag（GAAMA/ZeroMemory） | QUEUED | A 级卡；10-04 雷达增量，但论文线无主仓库；MemGraphRAG 已考古 |
| 8 | owasp-agentic-security / skill-supply-chain | COMPLETED | 卡内主力（owasp-agentic-skills 09-26、skillfortify 09-05）均已考古 |

## 排序理由（SOP 八维度 × browser-harness）
- **Knowledge Value**：A 级卡（评分在卡内）；"self-healing harness"是 2026 Agent 工程核心趋势
- **Agent-AI Engineering 相关性**：极高——CDP 直接控制 + LLM 运行时自生成 helpers/工具 + 无框架无 rails 的自愈范式，直接反哺 E2E-CLI 升级路线与 OpenClaw 浏览器操作场景
- **Novelty**：新范式（对照 Playwright 封装式预设边界，browser-harness 是"执行中动态补写"）
- **增长信号**：15.7k+ stars（2026-07 快照）、PyPI 0.1.6（2026-07-17）活跃更新
- **近期发布**：2026-04 起持续演进，browser-use 官方延伸
- **与已有 Corpus 连接**：browser-use（09-22 已考古）同厂；harness 族与 deepseek-harness/opencode 对照；EP-002 权限边界（自愈写代码=高风险权限）
- **是否已考古**：否（corpus projects/ 无 browser-harness）
- **refresh 需求**：无（initial）
- **Benchmark 学习价值**：中等规模仓库可完整走 EK Graph/聚合规则/盲重建全流程

## 状态区分
- NEW：browser-harness（本轮选中）
- REFRESH_REQUIRED：a2a-protocol-v1、mcp-2026-07-28-stateless（10-04 雷达重大增量）
- QUEUED：agentic-inference-infra、agentic-attack、agentic-graphrag、OpenCodeReview 等
- COMPLETED：owasp-agentic-security 卡内主力、skill-supply-chain 卡内主力
- IN_PROGRESS：无

## 附：10-04 雷达增量已归档至对应卡（a2a/mcp/agentic-graphrag），未新开卡；本 Manifest 依此决策
