# 06 Validation & Evidence

## 1. 验证方法

| 检查 | 方法 | 结果 |
|------|------|------|
| Source Truth | 盲重建：独立重读 repo（README/LICENSE/requirements/MemGraphRAG.py 关键段/Memory.py 全文/openie_openai/rerank/prompts/evaluation/utils/index.py/retrieval_dataset_test.py）后与 EK 比对 | ✅ 无事实错误 |
| Coverage | 子系统强制覆盖：管线编排/记忆结构/OpenIE/LLM 层/嵌入/重排/prompts/评估/配置/CLI | ✅ 30 EK 覆盖 10 子系统 |
| Causality | 每条 EK 的因果声明回溯实现 | ✅ |
| Flow | 七类流每 Edge 回溯 symbol/file/condition | ✅ |
| Abstraction | L3/L4/L5 独立判定（无 Synthesizer 自我论证） | ✅ 8 KO/4 CM/4 M 均受支撑 |
| Counterexample | 反例预算制（每 L3+ 候选 ≥3 攻击） | ✅ 见下 |
| Epistemic | Fact/Observation/Hypothesis/Pattern 标注核对 | ✅ C-01~C-08 诚实标注 |

## 2. 反例攻击记录（Counterexample Budget）

### 对 KO-01（三层记忆）的攻击
- **A1**：schema 层是否真的必要？→ 反例：`build_from_raw_openie_results` 明确支持"两层记忆（无 schema）"（Memory.py L149 docstring）——schema 是可选的归纳层，非强制 ✅ 收紧：KO-01 已写明"上层由下层归纳而来（LLM）"为默认路径，两层模式是例外
- **A2**：双向链接是否始终一致？→ 实测验证 passage→facts 与 fact→passages 双向一致 ✅
- **A3**：频率语义是否统一？→ schema.frequency=关联 facts 数，fact.frequency=distinct passages——语义不同，已分别标注 ✅

### 对 KO-02（冲突三阶段）的攻击
- **B1**：LLM 判定是否可绕过？→ 是：`conflict_min_confidence` 门槛 + 类型过滤（duplicate/none/uncertain 跳过）——判定可被配置放宽 ✅ 已写入 EK-06
- **B2**：分量过大怎么办？→ `conflict_resolution_max_component_size` 超限直接跳过为 unresolved（**静默不解析**）——这是"硬冲突可能残留"的失败路径 ✅ 已写入 EK-07
- **B3**：LLM 引用错误 ID？→ by_normalized 归一化匹配回退 + 匹配不到则丢弃该动作（**LLM 错误可被静默丢弃**）✅ 已写入 EK-07

### 对 KO-03（检索融合）的攻击
- **C1**：无 phrase 在图中？→ `assert sum(node_weights) > 0` 直接崩溃（非优雅降级）——崩溃点是防呆而非容错 ✅ 已写入 EK-22
- **C2**：dense 分数 0？→ min-max 归一化后 ×0.05，PPR 仍可运行（reset 全 0 时 igraph 行为未验证）→ C-05
- **C3**：eval 污染？→ C-04 ✅

### 对 KO-07（名实之辨）的攻击
- **D1**：是否过度解读"Multi-Agent"？→ 反例：IRCoT（一步一 thought 迭代）确实含 agentic 元素；README 的 "Multi-Agent" 也可能指"多 LLM 角色协同"。**判定：命名 vs 实现差距属实（无循环/无工具/无状态），但"误导"程度需审慎——研究论文常见角色化描述** ✅ 收紧：KO-07 表述为"名实差距"而非"虚假宣传"
- **D2**：eval 是否真的危险？→ 输入是自身 OpenIE 结果 + 可信语料，非公网服务；危险性是条件性的 ✅ 收紧：C-04 标注"条件性"

## 3. Epistemic 状态清单（最终）

| 对象 | 状态 | 依据 |
|------|------|------|
| EK-01/02（三层记忆/去重/频率） | **Fact** | 代码 + 实测（S4） |
| EK-03~29（管线机制） | **Fact**（实现精读） | 代码（S3） |
| EK-23（命名差距） | **Fact**（并列声明与实现） | README + 代码 |
| KO-01 | **Pattern**（S4 支撑） | 实测 |
| KO-02~08 | **Pattern**（S3 支撑） | 代码 |
| CM-1~4 | **Cognitive Model**（本项目验证，cross-project pending） | 支撑 KO |
| M-1~4 | **Methodology**（本项目导出） | 支撑 KO |
| C-01~C-07 | **Hypothesis/Observation/Tentative** | 未跨项目 |
| C-08 | **Benchmark Candidate** | 实测 |

## 4. 未实跑项（诚实标注）

- OpenIE / 冲突检测 / 冲突解析 / QA 均依赖 LLM API（openai/vllm）与 embedding 模型（torch 2.5.1 等）——本环境未安装，未实跑；相关 EK 全部标 S3（实现精读）而非 S4/S5。
- PPR 行为（igraph prpack）未实跑（依赖 python_igraph 0.11.8）。
- 结论均不依赖未实跑项——核心数据结构已实测，检索/冲突的机制分析以代码为据。

## 5. 质量指标

| 指标 | 值 |
|------|-----|
| Facts/Evidence | ~50（distinct 证据点） |
| EK | 33（links 100%，游离 0；含 Reconciliation 补 3 条） |
| KO | 8（aggregation_rule 100%） |
| CM/M | 4 / 4 |
| 反例攻击 | 10 次（A1-3/B1-3/C1-3/D1-2） |
| 判定 | PASS（含 2 处收紧 + Reconciliation 3 错误/4 遗漏补正，见独立验证报告） |
| 未实跑声明 | 3 项（LLM 管线/PPR/评估） |
