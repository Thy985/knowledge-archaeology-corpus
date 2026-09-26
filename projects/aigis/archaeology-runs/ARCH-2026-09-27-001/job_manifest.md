# Job Manifest — ARCH-2026-09-27-001

- **job_id**: ARCH-2026-09-27-001
- **date**: 2026-09-27
- **mode**: initial（每日最多 1 深度项目）

## 候选排序（按 SOP 排序标准，非 star 排名）

| 排 | 候选 | 卡等级 | 知识价值 | Agent-AI 相关性 | Novelty | 增长信号 | 与已有 Corpus 连接 | 可考古性 | 决策 |
|---|---|---|---|---|---|---|---|---|---|
| 1 | **Aigis（killertcell428/aigis）** | A（agent-firewall-runtime-defense） | 高（确定性 guardrail = "无 LLM 判定"路线） | 极高（agent 运行时治理） | 高（09-11 品类成形） | 强（pushed 09-19，PyPI v2.0.1） | **昨日 AST10 直接闭环**（威胁分类↔运行时防御实现）；guardian/ai-protector 同类对照；openclaw 栈 | ✅ 主仓库已核验 | **选中** |
| 2 | OWASP Agent Memory Guard | A（同卡） | 高（ASI06 官方参考实现） | 高 | 中（OWASP 同源） | 中 | AST10/OWASP 体系闭环 | ✅ | 待选 |
| 3 | Pipelock（luckypipewrench/pipelock） | A（同卡） | 高（流量扫描+签名 receipts） | 高 | 中 | 中 | AST09 receipts 直接对照 | ✅ | 待选 |
| 4 | AWS Dogwood（agent-formal-verification 卡） | A | 高（形式化验证治理语言） | 极高 | 高（08-06 开源） | 中 | AST10 EK-22 Policy Enforcement 呼应 | ⚠️ 需确认仓库路径 | 待选 |
| 5 | AgentGuard（0xrem/agentguard） | A（同卡） | 中高（Prompt/Tool/Command 三层） | 高 | 中 | 中 | 权限边界（EP-002） | ✅ | 待选 |
| 6 | agentic-benchmarks（品类卡） | A | 高（基准信任危机方法论） | 高 | 高（TB4.0 今日增量） | 强 | deepseek-harness/agentevals | ⚠️ 无单一仓库（多基准） | 不适合深度考古 |
| 7 | otel-genai / local-edge-llm（品类卡） | A | 中高 | 中高 | 高（今日增量） | 强 | trace-based-agent-eval | ⚠️ 标准/平台品类 | 不适合深度考古 |
| 8 | computer-browser-use（品类卡） | A | 中 | 高 | 中 | 中 | browser-use 已考古 | ⚠️ 品类卡 | 已部分考古 |

## 选中项目

- **project**: Aigis（Agent 运行时确定性防火墙 / Deterministic Agent Firewall）
- **repository**: https://github.com/killertcell428/aigis.git
- **mode**: initial
- **priority**: A（雷达 09-11 明确"确定性 guardrail 三选一实测应尽快排期"）
- **source_entry**: `[cand]agent-firewall-runtime-defense-2026.md`
- **reason**:
  1. **与昨日 AST10（ARCH-2026-09-26-001）形成"威胁分类 → 运行时防御实现"的一对一闭环**：Aigis 的检测项（MCP rug-pull、memory poisoning、indirect injection、exfil）逐一对应 AST01/02/05/07/09——Benchmark 学习价值本轮最高
  2. **确定性路线（No LLM. No heuristics）**：与已考古 guardian（LegionForge）AI Protector（szesnasty）同类但独立实现，可做同类对照（雷达明确排期的三选一实测）
  3. **增长信号强**：PyPI v2.0.1（雷达记录 v1.1.11 已升级）、主仓库 pushed 2026-09-19
  4. **一手核验通过**：主仓库 killertcell428/aigis（Apache-2.0、Python、54★）✅；PyPI pyaigis v2.0.1 ✅；注意雷达卡引用的 gaebalai/aigis-kr 为早期镜像（8★、pushed 2026-05-18），考古以主仓库为准
  5. 与 openclaw（campus_order 栈）、MCP（rug-pull 检测）生态直接相关
- **已考古状态**: 未考古（品类卡部分项目 guardian/ai-protector 已归档，Aigis 未）

## 雷达上下文（09-27 第 25 次运行）
- 无新卡、37 卡维持；3 条增量（agentic-benchmarks TB4.0/Terminal-World、local-edge-llm PAIR/Nemotron 3.5、otel-genai CloudWatch Omni/OpenObserve）
- 雷达"下一步"延续：silver-shield 威胁模型 / campus_order 权限模型 diff 审查 / Validation 编译器通用 agent 路线
