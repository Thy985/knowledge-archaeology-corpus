# 03 — Knowledge Layer（L1→L5，宽底座 + 窄尖顶）

## 分层纪律
- L1/L2 已在 02 Engineering Knowledge（EK-01..14）完整保留——**不因形成 KO 删除**。
- KO 必须声明 `aggregation_rule`（R1-R4）+ 簇内 EK + 解释范围扩大论证。
- **跨项目状态升级**：Guardian + Aigis = 同品类双项目互证 → 标注 `2-project corroborated`（仍未到 Principle/Law，L4 保留）。

---

## KO-01 — 确定性判定：把注入面从文本空间移到结构空间
- **L3 Pattern**（aggregation_rule: **R1 机制簇**，跨项目）
- 簇内 EK：EK-01（7 检查 fail-fast）、EK-02（JWT ACL）、EK-03（registry）、EK-05（regex 分级）；跨项目对照：Aigis-EK-01（L1-L7 管线）、Aigis-EK-02（可解释 MatchedRule）
- **论证**：Guardian 判定输入是**结构化工具调用**（tool_id/args/action/token），Aigis 判定输入是**归一化文本**——同一"确定性判定"目标、两种载体，共同点是**无 LLM、无启发式、判定可解释**。`2-project corroborated`：品类共同主线（雷达 #12 观察：确定性/密码学验证/可证明成主线）在双项目实现中独立成立。
- **L4 Cognitive Model**：**判定的可注入性取决于输入空间的结构化程度——结构化的字段无法被自然语言污染。**（The injectability of a verdict depends on the structure of its input space; structured fields cannot be polluted by natural language.）
- Epistemic：Pattern/Model（S3 实现 + S4 测试）；2-project corroborated，跨品类验证 pending。

## KO-02 — 审计防篡改 = hash chain 机制（双项目同构）
- **L3 Pattern**（aggregation_rule: **R2 因果链簇**，跨项目）
- 簇内 EK：EK-09（SHA-256 canonical + prev_hash + genesis + 写失败非致命）；跨项目对照：Aigis-EK-06（HMAC-SHA256 + frozen entry + sort_keys canonical）
- **论证**：两个独立项目（Guardian PostgreSQL 表链 vs Aigis 文件日志链）独立选择了**相同的防篡改结构**：canonical 序列化 + 密码学哈希 + 前条哈希链接 + genesis 起点 + **写失败不阻断被审计动作**。跨项目同构是"不可抵赖审计"模式的强证据。
- **L4 Cognitive Model**：**审计链的完整性取决于链头（genesis）与链边（prev_hash）的不可伪造性，而非存储介质。**（The integrity of an audit chain depends on the unforgeability of its head and edges, not its storage medium.）
- Epistemic：Pattern/Model（S3 + S4 测试双项目）；2-project corroborated。

## KO-03 — 安全热路径 fail-closed：配置/异常默认拒绝
- **L3 Pattern**（aggregation_rule: **R3 不变量簇**，跨项目）
- 簇内 EK：EK-11（misconfig/auth → halt）、EK-08（坏规则跳过不崩）；跨项目对照：Aigis-EK-17（scan/policy 异常 → exit(2) 阻断）、Aigis-EK-10（零依赖减少配置面）
- **论证**：两项目独立在"安全热路径"上选择 fail-closed：Guardian 配置缺失 → GUARDIAN_MISCONFIGURED halt（"never silently allow"）；Aigis 检测异常 → exit(2)（"blocking (fail-closed)"）。同时**审计路径**独立选择 fail-open（写失败不阻断）。**判定 fail-closed / 审计 fail-open** 的边界在双项目独立出现。
- **L4 Cognitive Model**：**安全系统的错误态必须是可见的拒绝，而不是静默的放行——审计可缺失，判定不可缺省。**（A security system's error state must be visible denial, not silent allowance—audits may lapse, verdicts may not.）
- Epistemic：Pattern/Model（S3 + S4）；2-project corroborated。

## KO-04 — 蜜罐/金丝雀：预期内永不触发的信号工具
- **L3 Pattern**（aggregation_rule: **R4 主题簇**）
- 簇内 EK：EK-10（guardian_canary）、EK-06（sequence 作为行为轨迹信号）、EK-08（自适应规则热加载信号）
- **论证**：Guardian 注册一个"合法但不该被调用"的工具作为蜜罐——触发即探测/幻觉证据，零误报（因为合法 agent 永不调用它）。同类对照：数据库 canary 行、蜜罐 token（诱饵凭证）、web 蜜罐页面。解释范围：任何"可信执行环境"的探测检测设计。
- **L4 Cognitive Model**：**在可信域中放置不可触发的合法诱饵，使探测行为成为可观测信号。**（Placing untriggerable legitimate decoys in a trusted domain turns probing into an observable signal.）
- Epistemic：Pattern（S3 实现 + S4 测试 test_canary 相关）；单项目（Guardian 仅一例）→ Cross-project validation pending。

---

## 分层检查
| 层 | 数量 | 检查 |
|---|---|---|
| L1/L2（EK） | 14 | 全保留于 02，含 links |
| L3 Pattern | 4（KO-01..04） | 均有同类对照；KO-01/02/03 双项目互证 |
| L4 Model | 4（随 KO） | 一句话稳定关系，可回溯 EK |
| L5 Methodology | 0 | 2 项目证据不足，不升（诚实） |

## 五问检查（每条 KO）
1. 项目消失后还有价值？→ 是（确定性判定/审计链/fail-closed/蜜罐均跨项目）✓
2. 能改变未来工程决策？→ 能（防线设计/日志设计/错误态设计）✓
3. "我认为"还是"项目证据"？→ 双项目代码+测试证据 ✓
4. 会在另一项目重现？→ 会（KO-01/02/03 已在第二项目独立重现）✓
5. 偶然还是结构性问题？→ 结构性（安全系统设计约束的推论）✓
