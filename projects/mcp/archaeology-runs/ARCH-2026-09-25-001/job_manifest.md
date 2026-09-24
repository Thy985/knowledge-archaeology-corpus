# Job Manifest — ARCH-2026-09-25-001

> 生成时间：2026-09-25（每日 Knowledge Archaeology Runtime SOP 定时触发）
> 阶段② Daily Archaeology Job Selection 产出

## 1. KnowlegeMap 当前状态（2026-09-25 读取）
- **HEAD**: `8f631c5`（radar watch 2026-09-25，第 23 次运行）
- **候选卡**: 37 张（无新卡）；今日雷达无新品类达建卡阈值
- **今日雷达增量（3 条，均非新卡）**：
  1. agent-memory：EnSIMem（arXiv 2609.27279，"记忆=证据库而非摘要库"，与 Validation 编译器同构）
  2. computer-browser-use：Claude for Chrome（浏览器内置 agent 三巨头齐集，BragJack 攻击面扩大）
  3. agentic-inference-infra：Agentic KV Cache 管理研究爆发（ThunderAgent/Sutradhara/IntentKV/KernelFlume/DualPath/Astera Leo X）
- **雷达"下一步"验证优先级**（均指向自研项目，非外部考古对象）：Tafcm 本地记忆设计 ＞ dsh 长会话 KV 成本模型 ＞ campus_order 浏览器 agent 权限边界

## 2. Corpus 已有状态（可写仓，main HEAD `aef17c4`）
- 已合并 17 项目：agentevals / agent-governance-toolkit / aigis / ai-protector / clear / deepseek-harness / dogwood / dsh-memory-evolve / evolver / guardian / letta-code / memgraphrag / omnigent / openclaw / opencode / rampart / skillfortify
- 在途 PR（Owner 决定合并，与本次无关）：#19 hermes-agent / #20 dynamo / #22 browser-use / #23 e2b / #24 a2a（昨日考古）
- **已考古卡对应**：deepseek-harness、rampart、skillfortify、letta-code、memgraphrag、opencode、dogwood、omnigent、a2a

## 3. 候选排序（按 Knowledge Value / Agent-AI 相关性 / Novelty / 增长信号 / 近期发布 / Corpus 连接 / 是否已考古 / Benchmark 价值；不按 star）

| # | 候选 | 卡级 | 未考古 | 增长信号 | 与 Corpus 连接 | Benchmark 价值 | 判定 |
|---|---|---|---|---|---|---|---|
| 1 | **MCP 2026-07-28 stateless（modelcontextprotocol）** | S | ✓ | 强：Registry 10k + Skills Final（SEP-2640 09-13 合并）+ 97M 月下载 + pushed 09-24 | **A2A（昨日）直接互补**、OpenClaw Skills、skillfortify | **最高**：验证 KO-07（A2A 水平/MCP 垂直分层）、C-07（扩展 metadata vs SEP-2640） | **选中** |
| 2 | OWASP Agentic Security（ASI01-10/AST01-10/Secure MCP） | S | ✓ | 中（2026-08 更新） | dsh-pentest/silver-shield/EP-002（自研）| 中（标准库，MCP 考古部分覆盖 Secure MCP 子项） | 下轮候选 |
| 3 | Agent Memory 2026 实现层 | S | 部分 | 中（EnSIMem 今日增量） | letta-code 已考古；增量指向自研 Tafcm 验证 | 中 | 无新代表仓库 |
| 4 | Agent Harness/Control Plane | S | 部分 | 中（JIT-Agent/HarnessDev 09-24 增量） | omnigent/deepseek-harness 已考古；StarHarness 过小 | 中 | 无新代表仓库 |
| 5 | PI-Hunter 注入审计线 | A | ✓ | 中 | dsh-pentest/TeamMind/Validation | 中 | arXiv 论文线，非代码仓库为主 |
| 6 | Agentic CLEAR（IBM） | A | ✓ | 弱 | silver-shield Benchmark/Tafcm ADI | 中 | 评估方法论，价值集中于论文 |
| 7 | computer-browser-use（Claude for Chrome） | A | 部分 | 强（今日增量） | browser-use PR#22 在途 | 中 | 边际递减 |

## 4. 选中项目
- **project**: MCP（Model Context Protocol）2026-07-28 stateless
- **repository**: `https://github.com/modelcontextprotocol/modelcontextprotocol.git`（Specification and documentation，9,297★，pushed 2026-09-24）
- **mode**: initial
- **priority**: S
- **reason**:
  1. S 级候选卡（最高优先）且从未考古、无在途 PR
  2. **Benchmark 学习价值最高**：与昨日 A2A 考古直接互补——A2A 是"水平协作协议"（agent↔agent），MCP 是"垂直工具协议"（agent↔tool）；一次考古可验证 KO-07（分层认知）与 C-07（扩展外置化趋势）两个跨协议假说，产出跨项目对照知识
  3. 增长信号最强：Registry 破万（9 个月 5 倍）+ 97M 月 SDK 下载 + Skills over MCP 正式定稿（SEP-2640，2026-09-13 合并）+ 主仓库 2026-09-24 仍在更新
  4. 近期发布：SEP-2640 Skills Final（09-13）+ 09-24 雷达增量（生态里程碑 + Skills 扩展定稿）
  5. 与已有 Corpus 连接：A2A（09-24）、OpenClaw（Skills 兼容体系）、skillfortify（skill 供应链治理）
  6. 为 Tafcm/agent-attention 的 Tool System 提供行业标准认知（卡中"潜在收益"）
- **source_entry**: `[cand]mcp-2026-07-28-stateless.md`（KnowlegeMap 03_expansion_queue/candidates/）

## 5. 考古对象边界
- 主对象：`modelcontextprotocol/modelcontextprotocol`（spec + docs + extensions + seps）——协议规范库，与 A2A 同构
- 参考（非主对象，按需佐证）：`modelcontextprotocol/servers`（参考实现集合，价值分散）、`go-sdk`（Tier1 SDK）
- 注意：MCP 2026-07-28 stateless 是**breaking change** 版本（取消 session、request 自带 `_meta`、MRTR、Mcp-Method 头路由等）——考古锚定当前 spec 版本（2026-07-28 + 后续 seps），需记录版本状态
