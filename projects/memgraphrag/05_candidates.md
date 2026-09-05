# 05 Candidates — 未确认内容

> 以下内容无法在本 run 确认，保留为候选。Hypothesis 不冒充 Fact。

## C-01 [Cross-project Hypothesis] "Multi-Agent" 命名与实现现实的系统性差距
- **内容**：README 自称 "Memory-based Multi-Agent System"，实现是 7 角色 LLM prompt 管道 + 多线程并发（无 agentic 循环/工具调用/状态 agent）。Hypothesis：2026 年"GraphRAG + Multi-Agent"命名热潮中，此类"多角色 prompt 管道"被广泛称为 Multi-Agent，需以代码核对。
- **证据**：README 标题 vs prompts/prompt.py + templates/ + ThreadPoolExecutor（本项目）
- **跨项目呼应**：skillfortify C-01（文档-实现矛盾）、RAMPART（observability 声称 vs 实现）
- **epistemic**: Hypothesis（单项目观察，跨项目 pending）
- **scope**: Cross-project，不确定

## C-02 [Tentative Pattern] 冲突处理三阶段分离（确定性候选 → LLM 判定 → 分量解析）
- **内容**：候选生成（结构规则）与判定（LLM）+ 解析（连通分量）分离，判定结果用归一化交叉匹配回源 ID。
- **是否可升 Pattern**：本项目已验证（S3）。跨项目是否通用待验证（尤其"归一化匹配回退"与"证据注入"是否在其他知识图谱项目出现）。
- **epistemic**: Pattern（本项目）/ Cross-project pending

## C-03 [Tentative Pattern] 图检索的显式回退（无图信号 → 稠密基线）
- **内容**：检索无 fact 时回退 dense passage retrieval。
- **跨项目呼应**：RAMPART 的"observability 降级"是不同域（可观测性 vs 检索）的同一思想（显式降级保底）。是否构成跨域模式待验证。
- **epistemic**: Pattern（本项目）/ 跨域呼应 pending

## C-04 [Potential Finding] eval() 解析内部生成内容的安全边界
- **内容**：rerank_facts 用 eval(fact_content) 解析 embedding store 中的 fact 内容；内容来源为 OpenIE（LLM）生成。若 OpenIE 输出被污染（prompt injection / 恶意语料），eval 可执行任意代码。影响面：研究工具（本地运行）风险中低，但作为"LLM 输出 → eval"反模式有普遍警示价值。
- **证据**：MemGraphRAG.py rerank_facts 两处 eval
- **epistemic**: Fact（代码存在 eval）/ Hypothesis（实际被利用性未验证）
- **scope**: 边界不确定——项目以可信语料 + 自身 LLM 为输入，非公网服务

## C-05 [Unresolved Contradiction] 并发与限流
- **内容**：多线程并发（memory_max_workers/retrieval_max_workers）与 LLM API 限流的关系未显式处理（无退避重试策略在 batch 层，只有 CacheOpenAI 的单次重试 max_retry_attempts=5）。是否在大语料上触发限流失败未验证。
- **证据**：ThreadPoolExecutor 多处 + config max_retry_attempts
- **epistemic**: Hypothesis

## C-06 [Scope-uncertain] 无自动化测试 vs KDD'26 基准声明
- **内容**：仓库无测试；README/论文声称基准结果（hotpotqa/musique 等）。基准可复现性（依赖/版本/seed）未在 repo 内完全锁定（requirements 有版本但无 lockfile）。
- **证据**：无 tests/ + requirements.txt（无 hash）
- **epistemic**: Observation
- **scope**: 不确定——研究仓库惯例 vs 可复现性标准

## C-07 [Tentative Pattern] LLM 输出解析的容错族（json_object + 异常回默认 + 归一化匹配）
- **内容**：response_format=json_object + _memory_json 容错 + by_normalized 匹配回退 + content-filter 最小化重试。该"解析容错族"是否可提炼为跨项目模式待验证。
- **证据**：EK-26/EK-07/EK-14
- **epistemic**: Pattern（本项目）/ Cross-project pending

## C-08 [Potential Benchmark Case] ThreeLayerMemory 纯 Python 定向实测
- **内容**：三层记忆构建（两层/三层）+ 去重 + 双向链接 + 序列化往返 已在无 LLM 依赖下实测通过。可固化为 knowledge-archaeology-skill 的跨项目 Regression Case：验证"数据结构级声明（docstring）与实现一致"。
- **证据**：本次实测脚本输出
- **epistemic**: Fact（本项目实测）

---

## C-09 [Observation] IRCoT 迭代检索为死代码 / 预留扩展
- **内容**：ircot_{dataset} 模板 + reason_step（qa_utils.py）实现完整但无调用者；max_qa_steps 配置定义了（默认 3）但核心类 0 引用。推测为论文实验（IRCoT 多跳）的预留/迁移残留，或未来版本接入。对"命名即能力"的提醒：模板/配置存在 ≠ 管线启用。
- **证据**：grep 全仓（reason_step 无调用者；MemGraphRAG.py 对 max_qa_steps 引用 0）
- **epistemic**: Observation（死代码状态确凿）/ Hypothesis（用途未验证）

---

## 候选处置

| 候选 | 处置 |
|------|------|
| C-01 | 保持 Hypothesis，跨项目验证 |
| C-02/C-03/C-07 | 保持 Tentative Pattern |
| C-04 | 保持 Finding（fact 部分）+ Hypothesis（利用性） |
| C-05 | 保持 Hypothesis |
| C-06 | 保持 Observation |
| C-08 | 提供为 Benchmark 候选（阶段 7 评估） |
| C-09 | 保持 Observation（死代码确凿） |
