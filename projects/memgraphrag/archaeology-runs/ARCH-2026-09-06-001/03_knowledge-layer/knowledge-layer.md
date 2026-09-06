# 03 Knowledge Layer — Generalized Knowledge（窄尖顶）

> 从 EK Graph 按聚合规则 R1-R4 聚簇。每个 KO 声明 aggregation_rule。未跨项目验证的一律标注 Cross-project validation pending。

## 聚合总览

| KO | 规则 | 簇内 EK | 主题 |
|----|------|---------|------|
| KO-01 | R1 机制簇 | EK-01/02/03/04 | 三层记忆组织 |
| KO-02 | R2 因果链簇 | EK-05→06→07 | 冲突感知构建管线 |
| KO-03 | R2 因果链簇 | EK-08→09→10 | 记忆派生图 + 检索融合 + fallback |
| KO-04 | R1 机制簇 | EK-13/15/17/25 | 并发 + 缓存 + 幂等工程 |
| KO-05 | R4 主题簇 | EK-19/20/24/29 | 评估即运行时的实验组织 |
| KO-06 | R4 主题簇 | EK-21/22/26 | 容错与防御工程 |
| KO-07 | R4 主题簇 | EK-12/23 | 设计-实现差距与安全边界 |
| KO-08 | R1 机制簇 | EK-09/11/28 | 图检索的信号融合 |

---

## KO-01 Pattern — 三层记忆组织（Schema/Fact/Passage 分层 + 双向链接）
- **aggregation_rule**: R1 机制簇。EK-01/02/03/04 共享"分层 + 索引"机制，跨 Memory/归纳/过滤 3 个实现面。
- **内容**：把语料组织成 本体层（类型三元组）→ 事实层（具体三元组）→ 原文层（chunk）三层，层间双向索引（1:N 与 N:M）；上层由下层归纳而来（LLM），下层是上层的可追溯证据。
- **证据**：EK-01（实测）/EK-02（实测）/EK-03/EK-04
- **epistemic**: Pattern。本项目实现 + 定向实测（S4）。
- **links**: [causal: →CM-1], [causal: →M-1]

## KO-02 Pattern — 冲突感知构建：确定性候选 + LLM 判定分离
- **aggregation_rule**: R2 因果链簇。EK-05→06→07 沿 causal 边形成完整链（候选生成→判定→解析）。
- **内容**：冲突处理三阶段分离：①结构规则确定性生成候选（同 subject+relation 异 object / 重复排除 / reverse-relation 可选）②LLM 角色判定（temperature=0，证据交叉匹配回 fact_id，置信度门槛）③连通分量批量解析（单次 LLM 处理整组冲突，优先级动作 kept<modified<discarded）。
- **证据**：EK-05/06/07
- **epistemic**: Pattern。本项目实现（S3，LLM 路径未实跑但逻辑精读）。
- **links**: [causal: →CM-2]

## KO-03 Pattern — 记忆派生图 + 图增强检索（phrase 权重 + PPR + fallback）
- **aggregation_rule**: R2 因果链簇。EK-08→09→10 形成"图构建→检索融合→回退"链。
- **内容**：冲突解析后的记忆生成 type/entity/passage 三类节点图（边带实体共现权重）；检索时 fact 短语赋权（频率归一化）→ dense passage 分数融合 → PPR 重排；无 fact 时显式回退纯 dense 检索。QA 侧为单步推理（IRCoT 迭代未接入主流程，见 C-09）。
- **证据**：EK-08/09/10/28
- **epistemic**: Pattern。本项目实现（S3）。
- **links**: [causal: →CM-3]

## KO-04 Pattern — 并发 + 缓存 + 幂等（研究管道的工程底座）
- **aggregation_rule**: R1 机制簇。EK-13/15/17/25 共享"并发执行 + 缓存复用 + 幂等重建"机制，跨 OpenIE/存储/LLM 3 面。
- **内容**：批量 LLM 调用用 ThreadPoolExecutor 两阶段并发；chunk 哈希 ID + 增量 OpenIE + force-from-scratch 标志支持增量/全量重建；LLM 响应缓存（cache_hit 统计）降本。
- **证据**：EK-13/15/17/25
- **epistemic**: Pattern。本项目实现（S3）。
- **links**: [causal: →M-3]

## KO-05 Pattern — 评估即运行时（gold 传入 + 指标即插即用）
- **aggregation_rule**: R4 主题簇。EK-19/20/24/29 围绕"实验组织与评估"覆盖互补维度。
- **内容**：QA/检索评估在运行时传入 gold 触发（EM/F1/Recall@k）；数据集特定 prompt 模板（缺失 fallback musique）；结果 JSON 含完整配置与逐 query 详情——实验可复现、可审计。
- **证据**：EK-19/20/24/29
- **epistemic**: Pattern。本项目实现（S3）。
- **links**: [causal: →M-3]

## KO-06 Pattern — 容错与防御工程（fallback 族）
- **aggregation_rule**: R4 主题簇。EK-21/22/26 围绕"失败路径处理"覆盖互补维度。
- **内容**：三类容错：①content-filter fallback（LLM 拒绝时用最小化 payload 保守重试）②JSON 解析容错（异常/空返回默认结构不崩溃）③防御性断言（空权重防 PPR 崩溃）+ assert False 作离线流程中断。
- **证据**：EK-21/22/26
- **epistemic**: Pattern。本项目实现（S3）。
- **links**: [causal: →CM-2]

## KO-07 Pattern — 设计命名 vs 实现现实（"Multi-Agent" 名实之辨）
- **aggregation_rule**: R4 主题簇。EK-23（命名差距）+ EK-12（eval 边界）围绕"声明与实现的关系"。
- **内容**：README 自称 "Multi-Agent System"，实现是 7 角色 LLM prompt 管道 + 多线程（无 agentic 循环）；另 eval() 解析内部生成内容存在未受控输入面。二者共同点：**对外声明（架构名/安全）与实现细节（角色编排/eval）存在差距，考古需独立核对**。
- **证据**：EK-23/EK-12
- **epistemic**: Pattern（本项目暴露）。
- **links**: [causal: →CM-4], [causal: →M-4]

## KO-08 Pattern — 检索信号融合（phrase 权重 + 归一化 + PPR）
- **aggregation_rule**: R1 机制簇。EK-09/11/28 共享"多信号加权融合"机制。
- **内容**：检索分数 = 图信号（fact phrase 权重，频率归一化防高频实体放大）+ 稠密信号（dense 分数 × 小权重 0.05）+ PPR 传播；LLM 重排或阈值过滤作为 fact 选择的最后一道闸。
- **证据**：EK-09/11/28
- **epistemic**: Pattern。本项目实现（S3）。
- **links**: [causal: →CM-3]

---

## L4 认知模型（Cognitive Model）

### CM-1 记忆系统的可信性来自"分层 + 可追溯"
**一句话**：把知识组织成"抽象层（schema）→ 事实层 → 原文层"并保持层间链接，任何上层结论都可下溯到原文证据，是可追溯检索的前提。
- 支撑：KO-01。本项目验证（S4 实测层间链接），跨项目 pending。

### CM-2 不确定认知与确定性结构分离
**一句话**：把需要判断力的事（冲突判定/本体归纳/QA 推理）交给 LLM（低温、JSON 约束、证据注入），把可以机械化的事（候选生成/图构建/频率计算）交给代码——两类的分离决定管线可靠性。
- 支撑：KO-02/KO-06。本项目验证（S3）。

### CM-3 图检索的"信号融合 + 显式回退"模式
**一句话**：图增强检索的价值在信号互补（结构 + 稠密），可靠性在显式回退（无图信号 → 纯稠密），不因图存在而放弃稠密基线。
- 支撑：KO-03/KO-08。本项目验证（S3），与 RAMPART 的「observability 降级」形成跨项目呼应（见 C-03）。注：QA 侧单步推理 + IRCoT 预留（C-09），「图检索增强」结论仅覆盖检索阶段。

### CM-4 对外命名与实现机制必须独立核对
**一句话**：项目自称的架构范式（"Multi-Agent"）与内部安全边界（eval）都不可直接采信，需以代码实现为准独立核对。
- 支撑：KO-07。本项目暴露（与 skillfortify 的文档-实现矛盾 C-01 同型，跨项目呼应，见 05 C-01）。

---

## L5 方法论（Methodology）

### M-1 三层记忆组织法
可操作准则：构建知识系统时按"抽象模式层 / 事实层 / 原文层"组织，上层归纳、下层溯源，层间保持双向索引；归纳过程用 LLM + 类型归一化，过滤用频率阈值（absolute 或 percentile）。

### M-2 冲突处理三阶段分离法
可操作准则：冲突/矛盾处理采用"确定性候选生成 → 低温 LLM 判定（带证据与置信度门槛）→ 连通分量批量解析（优先级动作 + 保守 fallback）"；判定结果必须能交叉匹配回源 ID（归一化匹配），不可只信 LLM 返回的引用。

### M-3 研究管道的可复现三件套
可操作准则：①中间产物全落盘 + force 重建标志（增量/全量）②LLM 响应缓存（cache_hit 统计）③评估随运行时传入 gold 执行（指标即插即用 + 完整配置入结果 JSON）。

### M-4 名实核对准则
可操作准则：考古/评估任何自称"Agent/Multi-Agent"的系统时，必须核对：是否有 agentic 循环？是否有工具调用？是否有状态 agent？若无，如实标注为"多角色 prompt 管道"；同理，任何 eval/动态执行点必须核对输入来源边界。

---

## 升维纪律检查

| 检查 | 结果 |
|------|------|
| 每条 KO 有 aggregation_rule | ✅ 8/8 |
| 无"同子系统=聚合理由"假聚合 | ✅ |
| 每个 L4 有支撑 KO | ✅ CM-1~4 |
| Cross-project 标注 pending | ✅ |
| Hypothesis 未冒充 Fact | ✅（C-01~C-06 在 05） |
| 目标配比 | Facts ~50 → EK 33 → KO 8 → CM 4 → M 4 ✅ |
