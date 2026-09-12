# 03 — Knowledge Layer（L1→L5，宽底座 + 窄尖顶）

## 分层纪律
- L1/L2 已在 02 Engineering Knowledge（EK-01..16）完整保留——**不因形成 KO 删除**。
- KO 必须声明 `aggregation_rule`（R1-R4）+ 簇内 EK + 解释范围扩大论证。
- 单一项目证据 → 最高 L3，标注 `Cross-project validation pending`；不写"普遍定律"。

---

## KO-01 — 确定性判定优于概率判定：可解释性是可审计性的前提
- **L3 Pattern**（aggregation_rule: **R1 机制簇**）
- 簇内 EK：EK-01（L1-L3 规则管线）、EK-02（MatchedRule）、EK-15（诚实边界声明）
- **论证**：同一"判定"问题，Aigis 选确定性规则（无 LLM、无启发式），每步产出可解释单元（rule_id/score_delta/owasp_ref）；对比 agentevals 的 LLM judge 谱系（已考古），同一领域出现两条路线。同类对照：静态分析（Semgrep）vs 模糊测试（LLM 风格）；签名杀毒 vs 行为启发式。解释范围：任何"安全判定"系统的设计取舍。
- **L4 Cognitive Model**：**当后果不可逆时，判定的可解释性优先于判定的覆盖面。**（When consequences are irreversible, the explainability of a verdict outranks its coverage.）
- Epistemic：Pattern/Model（S3 实现 + S4 测试 + S7 基准自述）；跨项目验证 pending。

## KO-02 — 纵深防御 = 不同失效模式的独立层（而非重复层）
- **L3 Pattern**（aggregation_rule: **R2 因果链簇**）
- 簇内 EK：EK-01→EK-03→EK-04→EK-05→EK-07→EK-08（输入归一→解码→taint→沙箱→规约→FSM）
- **论证**：L1-L7 每层对应**不同攻击面与失效模式**（文本注入/编码混淆/权限滥用/执行痕迹/属性违反/状态漂移），且每层显式"只补上层缺口"（L2 只查 L1 未命中类别）——是正交分层而非重复堆叠。同类对照：纵深防御（网络层/主机层/应用层）、多层验证（input validation + WAF + RASP）。
- **L4 Cognitive Model**：**防御层的价值在于失效模式的正交性，而非层数本身。**（The value of a defense layer is the orthogonality of its failure modes, not its count.）
- Epistemic：Pattern/Model（S3 实现 + ARCHITECTURE 明示分层动机）。

## KO-03 — 证据不可抵赖需要机制与测试双重保障
- **L3 Pattern**（aggregation_rule: **R3 不变量簇**）
- 簇内 EK：EK-06（HMAC+hash chain+frozen）、EK-14（alerts 永久保留）、EK-11（MCP 快照即证据）
- **论证**：安全日志"防篡改"不能只靠设计声明——Aigis 用 frozen dataclass（语言级）、HMAC 签名（密码级）、hash chain（结构级）三层，且 test_tampered_* 族直接断言"篡改任意字段→验证失败"。同类对照：区块链式审计、WORM 存储、签名的 CI 可追溯性。
- **L4 Cognitive Model**：**不可抵赖性是设计目标，也是测试断言——没有测试的防篡改只是愿望。**（Non-repudiation is both a design goal and a test assertion; tamper-proofing without tests is a wish.）
- Epistemic：Pattern/Model（S4 测试强证据：test_tampered_action/risk_score/outcome_fails_signature）。

## KO-04 — 安全工具的供应链与运行时同权（零依赖 + 可审计）
- **L3 Pattern**（aggregation_rule: **R3 不变量簇**）
- 簇内 EK：EK-10（零依赖哲学）、EK-11（MCP trust score）、EK-13（对抗反馈循环）
- **论证**：保护 Agent 的工具自身不能成为攻击入口——Aigis 用零核心依赖（供应链面最小化）+ MCP 工具信任评分（扩展面审查）+ 对抗循环（能力面持续更新）。同类对照：安全代理零依赖（scc 类工具）、最小攻击面原则、SBOM 治理。
- **L4 Cognitive Model**：**信任链的强度由最弱的依赖决定——安全工具的依赖面即其攻击面。**（A trust chain is as strong as its weakest dependency; a security tool's dependency surface is its attack surface.）
- Epistemic：Pattern/Model（S3 实现 + D8/D9 决策留档为证据）。

---

## 分层检查
| 层 | 数量 | 检查 |
|---|---|---|
| L1/L2（EK） | 16 | 全保留于 02，含 links |
| L3 Pattern | 4（KO-01..04） | 均有同类对照 + 跨项目 pending 标注 |
| L4 Model | 4（随 KO） | 一句话稳定关系，可回溯 EK |
| L5 Methodology | 0 | 单项目证据不足，不升（诚实） |

## 五问检查（每条 KO）
1. 项目消失后还有价值？→ 是（安全判定/纵深防御/审计/供应链四主题均跨项目）✓
2. 能改变未来工程决策？→ 能（防线设计/日志设计/依赖策略）✓
3. "我认为"还是"项目证据"？→ 项目证据（代码+测试+架构文档）✓
4. 会在另一项目重现？→ 会（任何 agent 安全/runtime 治理系统）✓
5. 偶然还是结构性问题？→ 结构性（架构决策的推论）✓
