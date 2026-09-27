# Job Manifest — ARCH-2026-09-28-001

## 候选排序（按 SOP 标准，非 star 排名）

| 排序 | 候选（候选卡） | Knowledge Value | Agent-AI 相关性 | Novelty | 增长/近期 | Corpus 连接 | 是否已考古 | 结论 |
|---|---|---|---|---|---|---|---|---|
| 1 | **AProver（agentic-prover/aprover）**〔agent-formal-verification-2026〕 | 高：LLM+BMC 混合验证 = "验证编译器"机器可核验形态 | 极高：Agentic Model Checking 范式 | 高：2026-05 创建，方法独特 | push 2026-09-15（活跃） | 强：Aigis（确定性防御）/AST10（威胁分类）/RAMPART（工具层） | 未考古 | ✅ **选中** |
| 2 | AWS Dogwood（agent-formal-verification-2026） | 高：运行时验证治理语言 | 极高 | 高：2026-08 开源 | 待核 | 强（同上） | 未考古 | ❌ 官方 repo 未定位（GitHub 搜索仅第三方引用，存在性存疑） |
| 3 | TencentCloudADP/youtu-graphrag（agentic-graphrag-2026） | 高：生产级 GraphRAG 栈 | 中：RAG 侧 | 中：ICLR 2026 | push 2026-02-26（近期信号弱） | 中：memgraphrag 品类对照 | 未考古 | 备选 |
| 4 | microsoft/agent-framework（agent-harness-control-plane） | 高：生产级 agent 框架 | 极高 | 中 | push 2026-09-27（极活跃） | 强：omnigent/deepseek-harness 对照 | 未考古 | 备选（大仓，次日可考） |
| 5 | agentic-attack-2026 | 高：威胁范式 | 极高 | 高（雷达 09-28 头号） | 事件密集 | 连接 silver-shield/dsh-pentest（内部） | 非仓库 | ❌ 事件/范式卡，无考古仓库对象 |

## 选中项目

- **job_id**: ARCH-2026-09-28-001
- **project**: aprover（BMC-Agent）
- **repository**: https://github.com/agentic-prover/aprover.git
- **mode**: initial
- **priority**: P1
- **reason**: ① 与最近三轮考古形成认知闭环——AST10（威胁分类）→ Aigis（运行时确定性防御）→ AProver（AI 生成代码的机器可核验证明）："攻击面→防御→可证明性"递进，每轮互为上下文；② AProver 的 *Agentic Model Checking*（arXiv 2605.21434：LLM 语义推理 + BMC solver 形式保证）是"验证编译器"（Claim→Operationalization→Evidence→Judgment）的方法论直接输入；③ soundness_policy 的"删除非对称性 + 三种判定类型"（deterministic_verifier / self_verifying_witness / agentic_judgment）与 Aigis 审计决策解耦、RAMPART 工具层形成三角互证；④ repo 规模适中（bmc_agent/ 73,867 行，268 py 文件），可完整深读；2026-09-15 仍在 push（增长信号真实）。⑤ 属 agent-formal-verification-2026 卡（G5 战略级），该卡另一对象 Dogwood 官方 repo 无法定位，AProver 是该线唯一可考古的具体仓库。
- **source_entry**: KnowlegeMap `03_expansion_queue/candidates/[cand]agent-formal-verification-2026.md`（AProver 条目 + GitHub agentic-prover/aprover）
