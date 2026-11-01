# Job Manifest — ARCH-2026-10-22-001

## 定案
- **job_id**: ARCH-2026-10-22-001
- **project**: agent-aegis（PyPI 包名 `agent-aegis`；源码仓库 **Acacian/aegis**）
- **repository**: https://github.com/Acacian/aegis.git
- **mode**: initial
- **priority**: P0
- **source_entry**: 03_expansion_queue/candidates/[cand]agent-aegis-governance-2026.md（Harness 周 Day2 增量扫描建卡）+ 07_radar_watch/current_focus.md（周五验证计划②：guardrail-策略引擎对比池 Aigis·MS AGT·agent-aegis）

## 候选排序（Harness/治理 周 Day4，2026-10-22）
| 排名 | 候选 | 依据（Knowledge Value / 相关性 / Novelty / 增长 / 近期发布 / 连接 / 已考古 / Benchmark） | 状态 |
|---|---|---|---|
| 1 | **agent-aegis（Acacian/aegis）** | 能力树缺口②直接命中（C2 无状态策略引擎 + D1 确定性 guardrail 一体化）；★18/Python/MIT；**pushed 2026-10-16（考古前 6 天活跃）**；created 2026-03-20；与 corpus 品类对照锚丰富（aigis/guardian/ai-protector 同品类 + agent-governance-toolkit 无状态策略对照 + Archon 10-21 控制面治理对照：内部 gate vs 外部中间件）；未考古；周五对比池成员（需源码级证据） | **NEW → 选中** |
| 2 | Warden（warden-mcp 0.2.1） | MCP drop-in 中间件（audit/policy/human-approval/injection defense），雷达 10-11 记 LobeHub/AlwaysReadyAllies——**GitHub 无该仓库、PyPI project_urls=None（无可考古源码）** | 排除（无源码） |
| 3 | Vaikora Guard MCP（Data443/vaikora-guard-mcp） | ★0、pushed 2026-05-26（与"10-15 开源"不符）、无第三方证据 | 排除（规模过小） |
| 4 | mcpkernel（piyushptiwari1/mcpkernel） | ★2、pushed 2026-06-16（与雷达 10-08 不符）；概念新颖但仅 PyPI 自述 | 排除（规模过小/证据不足） |
| 5 | agent-containment-breach-2026 集群 | 威胁情报多事件合成（非单项目仓库）；Day4 已升级证据线 4→6（Anthropic alignment + OpenAI Misalignment）；周五精读回填 E 层 | 非深考古目标 |
| 6 | Personal Agent Protocol（PAP/Poppy） | 10-21 fallback A 级新卡——**协议草案非代码仓库**；卡自标"领域外——非 Harness/治理深耕"，验证待 v0.1 规范 | 非仓库，移交 Runtime 周 |
| 7 | Strands Box / MXC / Archon | COMPLETED（10-19/10-20/10-21 考古过）——周五做对照与事件精读，非今日深考古 | COMPLETED 对照基准 |

## 选中理由（7 条）
1. **本周焦点**：Harness/治理 周 C 层（权限与策略治理）+ D 层（运行时防御）——agent-aegis 三支柱（runtime guardrails / YAML policy engine + approval gates / AGEF·AGP 开放标准）直接覆盖缺口②（无状态策略引擎/确定性 guardrail 对比）
2. **Novelty**：2026-03 创建、PyPI 0.4.0（10-15）→ 1.0.0 演进、**源码 pushed 10-16**——考古当日（10-22）前一周仍活跃，增长信号明确
3. **未考古**：corpus 35 项目无 aegis
4. **与已有 Corpus 连接**：aigis/guardian/ai-protector（同品类 contrast 锚）· agent-governance-toolkit（无状态策略引擎对照）· Archon（10-21：进程内控制面 gate vs 进程外 drop-in 中间件治理——同一目标两种解法）· dsh（approval seam）
5. **周五验证衔接**：guardrail-策略引擎对比池点名成员（Aigis·MS AGT·agent-aegis）——今日考古提供源码级证据，周五直接复用
6. **Benchmark 学习价值**：agent 框架插桩（LangChain/CrewAI/OpenAI/Anthropic 一行接入）+ 声明式审批门 + PII/注入阻断 = Gold Record 候选；"厂商自述 AGEF/AGP 标准无第三方证据"是 Epistemic 审计的天然靶点
7. **事件集群互文**：Archon CVE（控制面配置缺失）vs agent-aegis（运行时中间件防御）——"外因性配置缺失"与"运行时注入防御"的对照实证，为周五事件精读（E 层回填）补防御侧源码证据

## 排除说明
- Warden/mcpkernel/Vaikora：雷达 B 级记录的信息（仓库归属/日期）经 PyPI+GitHub 实测与卡不符或无公开源码——按"事实必须可追溯"原则排除，维持 B 级观察
- PAP：协议草案（无仓库），卡自标领域外，移交 Runtime 周深耕
