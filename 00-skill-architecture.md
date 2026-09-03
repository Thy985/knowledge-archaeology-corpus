# Knowledge Archaeology Skill 架构权威参考

> 本文档由 knowledge-archaeology skill 全部 44 个文件完整读取后固化，作为后续所有考古 Subagent 的统一执行依据。
> 生成时间：2026-09-03 | 适用项目：openai/codex

---

## 一、技能定位与第一原则

### 1.1 定位
Multi-Agent Knowledge Archaeology System —— 把软件项目（仓库/代码/文档/ADR/测试/Issue/RUN/审计）只读"考古"成可长期复用、可升维、可落飞书知识库的知识资产。

**核心命题**："这个项目让我们认识到了什么？"而非"这个项目有什么？"

### 1.2 第一原则
1. **仓库 = 工程事实/可执行资产；知识库 = 提炼的知识/认知资产**
2. **发现 ≠ 提炼 ≠ 升维 ≠ 验证**——四件事天然存在认知冲突，必须由不同角色承担，禁止同一角色自我确认
3. **Agent 之间只传递结构化 Artifact**，不传递长文本报告（否则退化为"token 接力"）
4. **只读**：禁止修改代码/文档/ADR/Issue/PR/git history
5. 所有下游 Agent 在 **Evidence Graph** 上推理，禁止"凭记忆整理"

---

## 二、五层知识阶梯（L1→L5 逐级晋升）

### 2.1 层级定义

| 层级 | 名称 | 核心问题 | 写法要求 | 反例（不合格） |
|------|------|---------|---------|---------------|
| L1 | 工程事实 | 发生了什么？ | 具体可溯源，带符号（文件:行号/ADR/commit/RUN） | "项目有 X 模块"（无来源） |
| L2 | 工程知识 | 为什么这样做？ | 因果解释 | "因为这样设计更好" |
| L3 | 工程模式 | 这类问题怎么解？ | 可复用模板 + 同类对照 | 只描述本项目 |
| L4 | 认知模型 | 背后的稳定关系？ | 一句话稳定关系 | 复杂绕口的理论 |
| L5 | 方法论 | 如何行动？ | 可操作准则（能做） | 抽象口号 |

### 2.2 晋升三条件（每升一级必须证明）
1. **New explanatory power**：解释范围确实扩大（不只是文字更抽象）
2. **Supporting evidence**：有证据支撑（Evidence Graph 中的节点）
3. **Scope validity**：解释范围扩大是合理的（单模块→项目 Pattern→通用原则，每步有依据）

### 2.3 降级规则
- L4 candidate 无法证明 scope validity → demote to L3
- 单一项目证据不升 L5 成"方法论"（需跨项目，否则标 `Cross-project validation pending`）
- **禁止跨级**（L1 直接跳 L5 = 臆想）

### 2.4 抽象层级（L0-L4，与五层阶梯对应）
| 层 | 描述 | 价值 |
|----|------|------|
| L0 | Project Fact（仅描述事实） | 低 |
| L1 | Implementation（具体实现） | 中 |
| L2 | Engineering Experience（经验） | 较高 |
| L3 | General Pattern（可迁移模式） | 高 |
| L4 | Principle（原则） | 最高 |

---

## 三、Agents 角色清单（20 个角色）

### 3.1 Discovery 阶段（发现）

| 角色名 | 文件路径 | 一句话职责 | 输入 Artifact | 输出 Artifact |
|--------|---------|-----------|--------------|--------------|
| repository-mapper | agents/repository-mapper.md | 建立仓库地图，按知识密度标记 A/B/C 级路径 | 整个仓库（只读） | Repository Map |
| code-analyst | agents/code-analyst.md | 从代码提取实现事实（架构/控制流/状态/抽象/边界/错误路径） | Repository Map A 级代码路径 | Code Evidence Pack |
| doc-analyst | agents/doc-analyst.md | 从文档/ADR 提取设计意图（≠ 实现事实） | Repository Map docs 路径 | Doc Evidence Pack（layer: design_intent） |
| test-analyst | agents/test-analyst.md | 测试 = 可执行知识，提取系统真正重视的行为 | Repository Map 测试路径 | Test Evidence Pack |
| failure-analyst | agents/failure-analyst.md | 找系统付出认知成本的地方（修复/回退/绕过/异常） | git history/regression/spikes | Failure Evidence Pack |
| authority-analyst | agents/authority-analyst.md | 权限/信任边界/执行权威/沙箱/bypass（Agent/runtime 项目） | 权限矩阵/gate 逻辑/边界代码 | Authority Evidence Pack |

### 3.2 Fusion 阶段（融合）

| 角色名 | 文件路径 | 一句话职责 | 输入 | 输出 |
|--------|---------|-----------|------|------|
| evidence-assembler | （内置于 Orchestrator） | 把多源证据融合成 Evidence Graph | 多个 Evidence Pack | Evidence Graph |

**融合规则**：
- ≥2 独立源 = Supported Fact
- 单源 = Unverified Observation（不进入下游）
- 冲突证据保留（标 `contradicts: <EV-id>`），不删除

### 3.3 Mining 阶段（挖掘）

| 角色名 | 文件路径 | 一句话职责 | 输入 | 输出 |
|--------|---------|-----------|------|------|
| problem-miner | agents/problem-miner.md | 找 Problem/Constraint/Pain/RootCause | Evidence Graph | Problem Graph |
| decision-miner | agents/decision-miner.md | 找 Decision/Alternative/Trade-off/Rejected | Evidence Graph（doc+code） | Decision Graph |
| pattern-miner | agents/pattern-miner.md | 只找重复结构（≥2 处独立出现），不得自行宣布原则 | Flow Atlas | Pattern Graph |
| flow-miner | agents/flow-miner.md | 构建六类 Flow（Control/State/Data/Evidence/Authority/Memory），必须锚定真实 symbol | Evidence Graph | Flow Atlas |

### 3.4 Synthesis 阶段（合成）

| 角色名 | 文件路径 | 一句话职责 | 输入 | 输出 |
|--------|---------|-----------|------|------|
| knowledge-synthesizer | agents/knowledge-synthesizer.md | 在各 Graph + Flow Atlas 上做 L1→L5 五层合成 | Evidence/Problem/Decision/Pattern Graph + Flow Atlas | Knowledge Objects |

### 3.5 Validation 阶段（验证）

| 角色名 | 文件路径 | 一句话职责 | 输入 | 输出 |
|--------|---------|-----------|------|------|
| truth-auditor | agents/truth-auditor.md | 这句话是真的吗？只能引用仓库证据 | Knowledge Package + Evidence Graph | verdict: PASS/FAIL/PARTIAL |
| coverage-auditor | agents/coverage-auditor.md | 还有什么重要东西没发现？独立重搜，不从已有清单猜 | Repository Map + Knowledge Objects | 遗漏清单 + Investigation Task |
| flow-auditor | agents/flow-auditor.md | Flow Atlas 是否真的对应代码？（抽验 symbol/edge/failure path） | Flow Atlas + 核心代码 | verdict: PASS/PARTIAL/FAIL |
| abstraction-auditor | agents/abstraction-auditor.md | L3/L4/L5 是否过度升维？（解释范围扩大是否合理） | Knowledge Objects（L3+） | verdict: OK/OVER-ABSTRACTED/UNDER-JUSTIFIED |
| counterexample-hunter | agents/counterexample-hunter.md | 专门找"哪里不是这样"（bypass/admin/test path/降级通道） | Knowledge Package 升维候选 + Flow/代码 | Counterexample 清单 |
| epistemic-auditor | agents/epistemic-auditor.md | 认知状态标注是否诚实（Hypothesis≠Validated≠Principle） | Knowledge Objects epistemic_status + 证据强度 | verdict: OK/OVERCLAIMED/UNDERSTATED |

### 3.6 Governance 阶段（治理）

| 角色名 | 文件路径 | 一句话职责 | 输入 | 输出 |
|--------|---------|-----------|------|------|
| reconciler | agents/reconciler.md | 显式处理多 Agent 冲突，产出条件化知识（四层分离） | 冲突的验证结果 + 各 Agent 证据 | 冲突化解记录 + 条件化知识包 |

### 3.7 角色激活裁剪规则

| 角色组 | 激活条件 |
|--------|---------|
| Code / Doc / Test Analyst | 总是激活（三视角基线） |
| Failure Analyst | 仓库存在修复史/大量 bugfix commit/regression 目录 |
| Authority Analyst | 项目含 Agent 系统/运行时权限模型/沙箱/插件执行 |
| Pattern / Flow Miner | 项目达到中等规模（多模块、有内核） |
| Counterexample Hunter | 已有升维候选（L3+），需要反例压力测试 |
| Epistemic Auditor | 知识库要落地（飞书/对外交付）时 |

> codex 项目判定：大型 Agent 系统（150+ crates），**全部角色激活**。

---

## 四、Contracts Schema 完整字段定义

### 4.1 Evidence Pack（证据契约）

所有 Discovery Agent 的产出必须是 Evidence Pack，不是自然语言报告。

```json
{
  "id": "EV-001",
  "stage": "code|doc|test|failure|authority|runtime",
  "discovered_by": "code-analyst",
  "source": "文件路径:行号 / ADR-NNNN / RUN-XXX / commit-hash",
  "claim": "只陈述观察，不做价值判断",
  "data_form": "数据形态变化（如 String → List<DocumentElement>）",
  "symbols": ["类名", "方法名", "文件名"],
  "strength": "S0|S1|S2|S3|S4|S5|S6|S7|S8",
  "detail": "关键原文引用",
  "timestamp": "YYYY-MM-DD"
}
```

**字段约束**：
- `source`：必须可追溯，禁止无来源断言
- `strength`：S0-S8 证据强度分级（见 4.5）
- `claim`：只陈述观察，"作者为什么这样"是 doc-analyst 的事
- `stage`：标记证据来自哪个认知视角

**Evidence Graph 融合规则**：
- ≥2 独立源支撑 = Supported Fact
- 单源 = Unverified Observation（不进入下游推理）
- 冲突证据保留在 Graph 中，标记 `contradicts: <EV-id>`

### 4.2 Knowledge Object（知识对象契约）

多 Agent 考古的最终产物。每个 KO 必须能回答"谁发现的、谁支持的、谁反对的、证据是什么、适用于什么"。

**核心字段（必填）**：

```json
{
  "claim": "一句话知识主张",
  "category": "PRODUCT|ARCHITECTURE|ENGINEERING|DESIGN|AGENT|RUNTIME|MEMORY|CONTEXT|PERMISSION|TESTING|EVALUATION|TOOLING|WORKFLOW|DECISION|FAILURE|EXPERIMENT|PATTERN|PRINCIPLE|METHODOLOGY",
  "abstraction": "L0|L1|L2|L3|L4",
  "value": "A|B|C|D|E",
  "provenance": {
    "discovered_by": "code-analyst",
    "supported_by": ["test-analyst", "failure-analyst"],
    "contested_by": ["counterexample-hunter"]
  },
  "epistemic_status": "Fact|Observation|Hypothesis|Validated Pattern|Principle|Law",
  "derivation": {
    "facts": ["EV-001", "EV-004"],
    "observations": [],
    "patterns": [],
    "models": []
  },
  "evidence": {
    "supporting": ["EV-001", "EV-004"],
    "contradicting": ["EV-012"]
  },
  "flows": {
    "control": "...",
    "state": "...",
    "data": "...",
    "evidence": "...",
    "authority": "...",
    "memory": "..."
  },
  "scope": {
    "applies_when": "适用条件",
    "does_not_apply_when": "不适用条件"
  },
  "confidence": "high|medium|low"
}
```

**可选字段**：

```json
{
  "disputes": [
    {"claim_by": "code-analyst", "counter_by": "counterexample-hunter", "resolution": "CONDITIONAL"}
  ],
  "validation": {
    "truth": "PASS|FAIL|PARTIAL",
    "coverage": "PASS|FAIL",
    "abstraction": "OK|OVER-ABSTRACTED",
    "counterexample": "none|found(详情)"
  }
}
```

**铁律**：
- `provenance` 是核心：必须知道谁发现、谁支持、谁反对
- `epistemic_status` 必须诚实：Hypothesis 不能写成 Validated Pattern
- `contradicting` 证据不能删除，只能通过 `disputes` 记录化解结果
- Abstraction 超过 L3 必须有 `validation.abstraction == OK`

### 4.3 Flow Schema（六类流契约）

flow-miner 的产出。每条流必须锚定真实符号，禁止概念箭头。

```json
{
  "flow_type": "control|state|data|evidence|authority|memory",
  "question": "这条流回答什么？",
  "chain": [
    {"node": "节点名", "symbol": "文件:行号", "role": "产生|验证|授权|执行|记录|守卫"}
  ],
  "gates": [
    {"name": "Gate 名", "logic": "判定逻辑", "symbol": "文件:行号"}
  ],
  "states": ["状态1", "状态2"],
  "conservation_points": ["守恒点描述"],
  "data_forms": ["String → AST"],
  "bugs": ["bug 落点描述"],
  "evidence": ["EV-001", "ADR-0008"]
}
```

**六类流定义**：

| 流类型 | 回答 | 关注 |
|--------|------|------|
| Control | 谁决定下一步？ | 决策点/分支/Gate/升级/回滚 |
| State | 状态如何变化？ | 状态容器/生命周期/可逆性/状态机 |
| Data | 数据从哪到哪？ | 转换边界/数据形态/守恒点/单一真相源 |
| Evidence | 声明如何变可证明？ | 证据生产链/Gate/分级/时效 |
| Authority | 每一步谁有执行权？ | 权限归属/执行边界/审计/bypass |
| Memory | 记忆如何沉淀？ | 分层上下文/状态指针/证据落盘 |

### 4.4 Validation Schema（验证契约）

6 类 Validator 的产出格式。

```json
{
  "validator": "truth-auditor|coverage-auditor|flow-auditor|abstraction-auditor|counterexample-hunter|epistemic-auditor",
  "target": "KO-007",
  "verdict": "PASS|FAIL|PARTIAL|CONDITIONAL",
  "findings": [
    {
      "issue": "问题描述",
      "evidence": "EV-012（证据引用）",
      "severity": "blocker|warning|info"
    }
  ],
  "recommendation": "降级到 L3 / 修改 claim 为条件化 / 补证据 / 接受"
}
```

### 4.5 证据强度分级（S0-S8）

| 级 | 含义 | 示例 |
|----|------|------|
| S0 | Opinion | 理论想法 |
| S1 | Discussion | 讨论 |
| S2 | Design Proposal | 设计提案 |
| S3 | Implemented | 已实现 |
| S4 | Test Validated | 自动化测试通过 |
| S5 | Runtime Validated | 真实运行验证 |
| S6 | Physical/Visual Validated | 物理/视觉验证 |
| S7 | Human Confirmed | 人工真实使用确认 |
| S8 | Repeated/Cross-project | 多项目都验证 |

**证据越强，结论越可升维。**

### 4.6 认知状态谱系（Epistemic Status）

```
Fact → Observation → Hypothesis → Validated Pattern → Principle → Law
```

| 状态 | 含义 | 证据要求 | 能否升维 |
|------|------|---------|---------|
| Fact | 项目内存在的事实 | S0-S3（有来源） | 可（作为 L1） |
| Observation | 单源观察 | 至少 1 个 Evidence | 需三角印证 |
| Hypothesis | 待验证假设 | 有理由相信 + 缺证据 | 需明确 Validation Path |
| Validated Pattern | 多源印证的模式 | S4+，≥2 独立源 | 可（作为 L3/L4） |
| Principle | 本项目验证的原则 | S5+，运行时/人工确认 | 可（作为 L4/L5） |
| Law | 普遍定律 | **S8 跨项目验证** | 否则不标 |

**跨项目规则**：未跨项目验证的 Principle 禁止表述为 Law，必须标 `Cross-project validation pending`。

### 4.7 价值评级（A-E）
- **A** 极高价值 / **B** 高价值 / **C** 一般 / **D** 项目局部 / **E** 不进知识库
- 评价维度：可迁移性 / 独特性 / 验证程度 / 长期有效性 / 认知增量 / 是否能指导未来决策

---

## 五、Protocols 协作协议

### 5.1 Agent Handoff（交接协议）

Agent 之间只传递结构化 Artifact，不传递自然语言长报告。

| 阶段→阶段 | 交接物 | Schema |
|-----------|--------|--------|
| Discovery → Fusion | Evidence Packs（多个） | evidence-schema.md |
| Fusion → Mining | Evidence Graph | evidence-schema.md |
| Mining → Synthesis | Problem/Decision/Pattern/Flow Graph | flow-schema.md |
| Synthesis → Validation | Knowledge Package | knowledge-schema.md |
| Validation → Reconciler | 冲突的 verdicts + 证据 | validation-schema.md |
| Reconciler → Final | 条件化知识包 | knowledge-schema.md |

**交接规则**：
1. 下游只读上游的 Artifact，不重新翻仓库（coverage-auditor 除外）
2. Artifact 必须完整——缺证据的 claim 不允许进入下游
3. 冲突的 Artifact 原样保留——不私下解决，交给 Reconciler
4. 每个 Artifact 带 provenance

### 5.2 Disagreement（分歧协议）

三步裁决：
1. **分类**：事实冲突（查证据强弱）/ 层面冲突（不选边，分层记录）/ 证据缺失（标 INCONCLUSIVE）
2. **分层**（Reconciler 使用）：
   ```
   Design Intent ≠ Implementation Reality ≠ Tested Behavior ≠ Runtime Observation
   ```
3. **输出**：事实冲突→依据证据强弱裁决；层面冲突→产出 CONDITIONAL 状态知识；证据缺失→INCONCLUSIVE

### 5.3 Escalation（升级协议）

**触发条件**：
1. 信息不足→启动对应 Analyst 补证据
2. 反例触发降级→Counterexample Hunter 发现不成立→Synthesizer 降级
3. 冲突无法化解→标 CONDITIONAL 或升级到人类
4. 盲区发现→Coverage Auditor 报新盲区→启动新 Investigation Task

**规则**：Orchestrator 只做调度决策，不亲自分析；升级循环有上限。

### 5.4 Evidence Provenance（证据溯源协议）

溯源链：`discovered_by → supported_by → contested_by → evidence`

规则：
1. 每个 EV 必须标 `discovered_by`
2. KO 必须标 `supported_by` / `contested_by`
3. 证据不可删除——即使被推翻，原证据保留并标记 `superseded_by`
4. source 必须可回看——文件:行号/ADR-NNNN/RUN-XXX，禁止"我记得"

### 5.5 Promotion（晋升协议）

晋升链：
```
L1 Candidate → Validator → L2 Deriver → Validator → L3 Pattern Miner → Validator
→ L4 Model Builder → Validator → L5 Methodology Candidate → Validator
```

每级晋升必须证明三条件（New explanatory power / Supporting evidence / Scope validity）。

关键门槛：
- L1/L2 由 Evidence Graph 直接支撑
- L3 需要 pattern-miner 的 Observed repeated structure（≥2 处独立出现）
- L4 需要 abstraction-auditor 的 OK 判定
- L5 需要 L4 稳定 + epistemic-auditor 的状态诚实

---

## 六、Workflows 执行流程与质量门

### 6.1 Archaeology Workflow（考古主流程）

**阶段 0 · 准备**：边界声明（只读）→ 角色激活（裁剪规则）→ repository-mapper 建仓库地图

**阶段 1 · Discovery**（多视角独立观察，并行）：
- code-analyst → 实现事实
- doc-analyst → 设计意图（≠ 实现事实）
- test-analyst → 可执行知识
- failure-analyst → 认知成本
- authority-analyst → 权限/信任边界/bypass
- 产出：多个 Evidence Pack

**阶段 2 · Evidence Fusion**：evidence-assembler 融合成 Evidence Graph
- ≥2 独立源 = Supported Fact
- 单源 = Unverified Observation（不进入下游）
- 冲突证据保留（标 contradicts）

**阶段 3 · Mining**（分别提炼不同东西）：
- problem-miner → Problem Graph
- decision-miner → Decision Graph
- pattern-miner → Pattern Graph（只报重复结构，不宣布原则）
- flow-miner → Flow Atlas（六类流，锚定真实符号）

**阶段 4 · Synthesis**：knowledge-synthesizer 在 Graph + Flow Atlas 上做 L1→L5 五层合成，产出 Knowledge Objects。**输入是 Graph，不是"仓库 + 记忆"。**

**阶段 5 · Validation**（多 Validator 交叉验证）：
- truth-auditor → 事实核验
- coverage-auditor → 独立重搜找盲区
- flow-auditor → Flow 对应代码？
- abstraction-auditor → 升维是否过度
- counterexample-hunter → 找"哪里不是这样"（可能触发降级）
- epistemic-auditor → 认知状态诚实？

**阶段 6 · Reconcile**：reconciler 显式处理冲突，产出条件化知识（design/impl/tested/runtime 四层分离）

**阶段 7 · Quality Gate + 反馈控制**：
```
全部 Validator PASS → Accept → Final Package
任一 blocker       → Revise → 回到对应阶段 Re-analysis
```
**这不是一次性生成报告，而是带反馈控制的认知 Runtime。**

**阶段 8 · 交付**：Knowledge Package（含 provenance / epistemic_status / 五层阶梯）+ 汇报格式（统计 / Top 高价值 / Top 跨项目模式 / Most Important Finding）

### 6.2 Validation Workflow（质量验证流程）

验证矩阵：

| Validator | 维度 | 关键问题 |
|-----------|------|---------|
| truth-auditor | 事实 | 这句话是真的吗？（只引用仓库证据） |
| coverage-auditor | 覆盖 | 还有什么没被发现？（独立重搜） |
| flow-auditor | 流 | Flow Atlas 对应代码吗？（抽验 symbol） |
| abstraction-auditor | 升维 | L3/L4/L5 过度升维吗？ |
| counterexample-hunter | 反例 | 哪里不是这样？（bypass/admin/test path） |
| epistemic-auditor | 状态 | 认知状态诚实吗？ |

逐项验证流程：对每个 KO 依次做 truth → coverage → flow → abstraction → counterexample → epistemic。

质量门：全部 PASS → Accept；任一 blocker → Revise → Re-analysis。

### 6.3 Benchmark Workflow（基准测试）

用已完成考古的项目验证 skill 本身质量。测量指标：增量发现 / 反例质量 / 降级有效性 / 冲突价值 / 证据支撑比例。

---

## 七、推荐 Wave 划分与 Artifact 传递链路

基于 codex 项目（大型 Agent 系统，150+ crates）的推荐执行 Wave：

### Wave 0：仓库测绘
- **角色**：repository-mapper
- **产出**：`01-repository-map.md`（A/B/C 级路径清单）
- **传递**：→ 所有 Discovery Agent

### Wave 1：Discovery（多视角并行证据提取）
- **角色**：code-analyst / doc-analyst / test-analyst / failure-analyst / authority-analyst
- **产出**：5 个 Evidence Pack → `02-evidence-packs/`
- **传递**：→ evidence-assembler（融合）

### Wave 2：Evidence Fusion + Mining
- **Fusion**：evidence-assembler → Evidence Graph → `03-evidence-graph.md`
- **Mining**：problem-miner / decision-miner / flow-miner / pattern-miner
- **产出**：Problem Graph / Decision Graph / Flow Atlas / Pattern Graph → `04-mining-graphs/`
- **传递**：→ knowledge-synthesizer

### Wave 3：Synthesis（五层合成）
- **角色**：knowledge-synthesizer
- **产出**：Knowledge Objects（L1→L5）→ `05-knowledge-objects/`
- **传递**：→ 所有 Validator

### Wave 4：Validation + Reconcile
- **角色**：truth / coverage / flow / abstraction / counterexample / epistemic auditors + reconciler
- **产出**：验证报告 + 条件化知识包 → `06-validation/` + `07-final-knowledge-package.md`
- **质量门**：全部 PASS → Accept；blocker → Revise 回对应 Wave

---

## 八、最终交付物清单与格式要求

### 8.1 交付物清单

| 序号 | 产物 | 格式 | 路径 |
|------|------|------|------|
| 0 | 技能架构参考 | Markdown | `00-skill-architecture.md` |
| 1 | 仓库地图 | Markdown | `01-repository-map.md` |
| 2 | Evidence Packs | JSON/Markdown | `02-evidence-packs/` |
| 3 | Evidence Graph | Markdown | `03-evidence-graph.md` |
| 4 | Mining Graphs（Problem/Decision/Flow/Pattern） | Markdown | `04-mining-graphs/` |
| 5 | Knowledge Objects | JSON/Markdown | `05-knowledge-objects/` |
| 6 | Validation 报告 | Markdown | `06-validation/` |
| 7 | 最终知识包 | Markdown | `07-final-knowledge-package.md` |

### 8.2 格式要求
- 所有 Evidence Pack / Knowledge Object 遵循 contracts schema（JSON 结构，可嵌入 Markdown）
- 每个 claim 必须带 `source`（文件:行号）
- 每个 KO 必须带 `provenance`（discovered_by / supported_by / contested_by）
- 每个 KO 必须带 `epistemic_status`（诚实标注）
- L3+ 必须带 `validation.abstraction` 判定
- 冲突证据不删除，通过 `disputes` 记录
- 最终汇报包含：统计 / Top 高价值知识 / Most Important Finding / 剩余风险

---

## 九、codex 项目专属判定

- **项目类型**：Agent 系统 / CLI 工具 / 运行时（Rust，150+ crates）
- **激活角色**：全部 20 个角色（大型 Agent 系统，含沙箱/权限/执行边界）
- **重点关注区域**：
  - `codex-rs/core/`：核心抽象
  - `codex-rs/exec/`、`codex-rs/sandboxing/`、`codex-rs/linux-sandbox/`：执行与沙箱（Authority Flow 重点）
  - `codex-rs/tools/`、`codex-rs/mcp-server/`：工具与 MCP
  - `codex-rs/memories/`、`codex-rs/history/`：记忆与历史（Memory Flow 重点）
  - `codex-rs/model-provider/`、`codex-rs/config/`：模型与配置
  - `docs/`：设计文档
  - `AGENTS.md`：Agent 治理文档
  - `skills/`：技能系统
- **知识密度预判**：沙箱/执行边界/权限模型/记忆系统/工具调用链 = 高知识密度区域
