# 03 — Knowledge Layer（L1→L5，宽底座 + 窄尖顶）

## 分层纪律
- L1/L2 已在 02 Engineering Knowledge（EK-01..16）完整保留——**不因形成 KO 删除**。
- KO 必须声明 `aggregation_rule`（R1 机制簇 / R2 因果链簇 / R3 不变量簇 / R4 主题簇）+ 簇内 EK + 解释范围扩大论证。
- 单一项目证据 → 最高 L3，标注 `Cross-project validation pending`；不写"普遍定律"。

---

## KO-01 — 记录一次、反复评测（Evaluation/Execution 解耦）
- **L3 Pattern**（aggregation_rule: **R2 因果链簇**）
- 簇内 EK：EK-01→EK-02→EK-03→EK-04（trace 采集 → 归一 → 可反复评测）
- **论证**：执行昂贵且非确定，评测廉价且可重复——解耦使"一次采集、无限次评判、零额外 token"成立。同类对照：录像回放复盘（sports/incident）、数据库 binlog 重放、录制回放测试（Puppeteer Replay）。解释范围从"评测"扩大到"任何以轨迹为证据的复盘"。
- **L4 Cognitive Model**：**执行是采样，评测是审阅——证据一旦落盘，评判与事实分离。**（Execution samples; evaluation reviews——once evidence is captured, judgment decouples from the fact.）
- Epistemic：Pattern/Model（本项目强证据 S4 测试 + S3 实现 + README S7 级自述；跨项目验证 pending → L3/L4 成立，L5 不写）

## KO-02 — 归一化优先于适配（格式薄层 + 注册表）
- **L3 Pattern**（aggregation_rule: **R4 主题簇**）
- 簇内 EK：EK-02（loader 注册表）、EK-04（消息双 schema + 三通道）、EK-05（格式优先级协议）、EK-11（sink 注册表）、EK-12（runtime 注册表）
- **论证**：系统面对 N 个外部格式/通道/消费目标时，不写 N² 适配，而是"每新增一个 X 只加一个注册表条目"。同类对照：编译器前端（多语言 → 统一 IR）、ORM（多 DB 方言）、Otel Collector（多 receiver→统一 pipeline）。解释范围扩大：任何"多外部面"系统。
- **L4 Cognitive Model**：**外部世界的多样性应在边界处收敛，不在内核中扩散。**（External diversity converges at the boundary, never in the core.）
- Epistemic：Pattern/Model（S3 实现 + S4 测试）；跨项目 pending。

## KO-03 — 确定性门禁与概率性评判分属不同信任层级
- **L3 Pattern**（aggregation_rule: **R3 不变量簇**）
- 簇内 EK：EK-14（信任谱系）、EK-06（ADK 双类指标）、EK-08（确定性结果身份）、EK-15（遥测保真度边界）
- **论证**：CI 门禁必须可复现（EXACT trajectory 确定性 pass/fail）；语义质量允许概率（LLM judge）——两者在同一框架共存但**永不混淆**（无 golden → error，不降级为 judge 猜测）。同类对照：CI 单元测试（确定性）vs 视觉回归截图 diff（阈值）；单元测试 vs 模糊测试。
- **L4 Cognitive Model**：**可复现性是门禁的前提，评判的置信度必须与判定机制匹配。**（Reproducibility is the precondition of a gate; the confidence of a verdict must match its mechanism.）
- Epistemic：Pattern/Model（S4 测试强证据）。

## KO-04 — 安全注入走"fail-closed 显式通道"而非环境魔法
- **L3 Pattern**（aggregation_rule: **R3 不变量簇**）
- 簇内 EK：EK-07（credential 私有 seam + fail-closed）、EK-08（确定性去重）、EK-09（409 冲突显式暴露）
- **论证**：凭据/冲突类敏感路径一律显式失败而非静默降级：credential_ref 未解析 → error；同 run_id 不同 spec → 409 返回已持久化 run 供 reconcile。同类对照：fail-closed 防火墙规则、k8s admission webhook deny-by-default。
- **L4 Cognitive Model**：**在不确定处失败，好过在错误处成功。**（Failing at the point of uncertainty beats succeeding through the wrong path.）
- Epistemic：Pattern/Model（S4 测试：test_credential_injection.py 明确覆盖 fail-closed 与并发隔离）。

---

## 分层检查
| 层 | 数量 | 检查 |
|---|---|---|
| L1/L2（EK） | 16 | 全保留于 02，含 links |
| L3 Pattern | 4（KO-01..04） | 均有同类对照 + 跨项目 pending 标注 |
| L4 Model | 4（随 KO） | 一句话稳定关系，可回溯 EK |
| L5 Methodology | 0 | 单项目证据不足，不升（诚实） |

## 五问检查（每条 KO）
1. 项目消失后还有价值？→ 是（可迁移到任何评测/复盘系统）✓
2. 能改变未来工程决策？→ 能（评测架构/凭据处理/门禁设计）✓
3. "我认为"还是"项目证据"？→ 项目证据（代码+测试+文档）✓
4. 会在另一项目重现？→ 会（dsh 评测、任何 agent harness）✓
5. 偶然还是结构性问题？→ 结构性（架构决策的推论）✓
