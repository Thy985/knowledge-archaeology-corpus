# 05 Candidates — 未确认假说与跨项目候选

> 所有不能确认的内容留在此处，**不写入已验证知识**。Hypothesis 不冒充 Fact；Cross-project Candidate 不写成已验证 Principle。

## C-01（NEEDS_HUMAN_REVIEW）Formal-Foundations "without over-approximation" 是措辞错误还是 soundness 声明过度
- **类型**: Hypothesis（文档-实现差距）
- **内容**: Formal-Foundations Theorem 2 声称能力推断 "without over-approximation or under-approximation"（exact）；实现明确是 conservative over-approximation。**over-approximation 恰是 soundness 的正确来源**，所以文档结论方向对、措辞错；但需要确认论文正文（arXiv 2603.00195）是否也这样写——若论文同样声称 exact，则属系统性声明过度。
- **证据**: wiki/Formal-Foundations.md（Theorem 2 段落）vs src/skillfortify/core/analyzer/engine.py L153-158
- **影响**: 若论文/文档一致声称 exact 而实现是 over-approximation，用户可能误解 soundness 的机制（误以为"精确分析"而非"有界近似"），且可能低估 4 类启发式之外的资源盲区（见 C-03）。
- **处置**: 标记 NEEDS_HUMAN_REVIEW；不自动升维为"文档失准"原则（单例）。

## C-02（Cross-project Candidate）CM-2 权限核对 + 权限单调 与 RAMPART 权威边界的关系
- **类型**: Cross-project Hypothesis
- **内容**: SkillFortify 的"声明-实际核对 + join 单调不放大"（CM-2）与 RAMPART（昨日考古）的 authority boundary 可能是同一跨项目模式的两种实例："Agent 相关系统的权限可信性来自显式边界 + 单调性"。但两项目领域不同（静态分析器 vs 运行时安全测试），需第三个项目佐证。
- **证据**: SkillFortify CM-2（本 run）+ RAMPART authority 边界（ARCH-2026-09-04-001）
- **状态**: Tentative pattern（跨项目，未验证）；可进入 corpus 的 cross-project 观测区。

## C-03（Scope-uncertain）能力推断只覆盖 4 类资源，soundness 声明范围可能被放大
- **类型**: Hypothesis
- **内容**: `_infer_capabilities` 仅推断 network/shell/environment/filesystem 4 类资源（EK-35）。但 Formal-Foundations 声称 Theorem 2 覆盖 "all capability-level threats"、"only access the resources and perform the actions it declares"。若 skill 声明涉及 4 类之外的资源（如 GPU/传感器/其他进程），推断盲区未在文档标注。
- **证据**: analyzer/engine.py L162-191（仅 4 类启发式）vs Formal-Foundations Theorem 2 措辞
- **影响**: soundness 的实际范围 = 4 类资源上的保守近似；文档"all capability-level"措辞扩大声明。需确认论文是否声明资源全集边界。

## C-04（Unverified）SAT 解析器（Theorem 4 Resolution Soundness）未实测
- **类型**: Hypothesis（实现存在性已确认，soundness 声明未验证）
- **内容**: core/dependency 有 resolver，python-sat 是可选依赖（本环境未安装）。Theorem 4 声称 SAT 求解同时满足版本/冲突/安全约束。未运行真实 SAT 求解验证。
- **证据**: pyproject.toml optional-dependencies（sat）；wiki/Formal-Foundations.md §SAT
- **处置**: 留给未来 run（装 python-sat 后补 e4_resolution 实验）。

## C-05（Scope-uncertain）registry 扫描（install-time 攻击缓解）实现深度未验证
- **类型**: Hypothesis
- **内容**: Formal-Foundations 说 v0.3.0 起 registry scanning 部分缓解 install-time 攻击（6% recall gap 的对应项）；CHANGELOG 0.5.0 说 registry fetches 有加固（redirect/public-address/size caps）。但 registry_cmd 的实际能力边界（能扫哪些 registry、如何评价安装前 skill）未逐行验证（httpx 可选未装）。
- **证据**: wiki/Formal-Foundations.md（v0.3.0 提及）；CHANGELOG 0.5.0；cli/registry_cmd.py（未深读）

## C-06（Fact 层已确认，处置在 KnowlegeMap）候选卡许可证记录与实际不符
- **类型**: Fact（discrepancy）
- **内容**: 候选卡 `[cand]skill-supply-chain-security-2026` 记录 MIT；实际 pyproject license=Elastic-2.0。本 run 已确认事实（EK-29），但**候选卡修正属于 KnowlegeMap 写操作，本 run 不写 KnowlegeMap**。
- **处置**: 留待 KnowlegeMap owner（或下个 Job Selection 阶段顺带更新扫描日志注记）。

## C-07（Unverified）测试覆盖深度
- **类型**: Observation
- **内容**: 182 测试文件、测试命名覆盖反规避/双用途/anti-leakage/信任性质；但 analyzer 核心模式（patterns.py 数百条模式）的逐模式测试覆盖比例未统计。
- **证据**: tests/ 结构；未跑 pytest 全量（避免环境副作用——本 run 只做定向实测）

## C-08（Tentative）性能规模与真实扫描效果未实测
- **类型**: Observation
- **内容**: 52k 行 Python / 22 框架，但真实仓库扫描的端到端性能（e7_endtoend 实验）与 SkillFortifyBench 540-skill 的 F1=96.95% 未在本环境复跑（benchmark corpus 在 companion repo skillfortifybench，非本 repo）。
- **证据**: README（SkillFortifyBench F1 96.95% 为文档声称）；benchmarks/experiments/（存在未跑）

---

## 候选处置汇总

| ID | 类型 | 状态 | 去处 |
|----|------|------|------|
| C-01 | Hypothesis | NEEDS_HUMAN_REVIEW | 06 验证 + 汇报给 owner |
| C-02 | Cross-project Hypothesis | pending 第三方 | corpus cross-project 观测 |
| C-03 | Hypothesis | scope 收紧 | 影响 soundness 声明范围解读 |
| C-04 | Hypothesis | 待补实验 | 未来 run |
| C-05 | Hypothesis | 待补实验 | 未来 run |
| C-06 | Fact | 已确认 | KnowlegeMap 侧处置（本 run 不写） |
| C-07 | Observation | 记录 | 本 run |
| C-08 | Observation | 记录 | 本 run（benchmark 复跑在 companion repo） |

## C-09（Observation，Reconciliation 补充）is_safe 二值语义混合（pattern_match vs capability_violation）
- **类型**: Observation
- **内容**: AnalysisResult.is_safe 是二值，但聚合两类 finding：Phase2 pattern_match（召回型，任何外部 URL 即报 data_exfiltration HIGH）与 Phase3 capability_violation（sound 型）。实测良性 skill（declared network:read + 1 个 URL）safe=False（仅因 pattern_match），无能力违反——用户看到 safe=False 无法直接区分"检测到疑似信号"与"能力越界"。
- **证据**: 定向实测（良性 URL 案例）；analyzer/models.py AnalysisResult
- **影响**: 报告层已有 finding_type 区分（pattern_match/capability_violation），但 is_safe 聚合二值可能误导；建议 UI/报告层显式拆分两类"不安全"语义。
- **处置**: 记录为 Observation；若跨项目复现可作为 UX/报告设计模式候选。
