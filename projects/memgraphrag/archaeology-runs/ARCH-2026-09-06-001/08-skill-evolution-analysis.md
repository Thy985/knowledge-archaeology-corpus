# Skill Evolution Analysis — ARCH-2026-09-06-001 (memgraphrag)

> 目标：分析本次真实项目考古暴露的 knowledge-archaeology-skill 系统性缺陷。不立即修改生产 Skill；先产出 Proposal + 候选变更 + 验证。

## 缺陷清单（按八类）

### EVID-01 [Evidence Failure · P0]
- **failure_id**: EVID-01 ｜ **type**: Evidence Failure ｜ **project**: memgraphrag ｜ **run_id**: ARCH-2026-09-06-001
- **observed_behavior**: 考古第一阶段把 `ircot_{dataset}` 模板存在断言为"IRCoT 迭代推理用于 QA"（EK-19），独立审计发现 `reason_step`（qa_utils.py L34-56）无调用者、`max_qa_steps` 在核心类 0 引用——该能力是死代码，实际 QA 为单步推理。
- **evidence**: qa_utils.py reason_step 全仓无调用者（grep）；MemGraphRAG.py 对 max_qa_steps 引用 0；qa() L1240-1330 为单次 LLM 调用 + split('Answer:')[1]
- **root_cause**: doc-analyst/flow-miner 以"模板/配置存在"为证据，未验证**调用图**（call graph）；Truth/Flow Auditor 无机械化"符号消费验证"（声明存在 ≠ 被调用）
- **impact**: **False Acceptance**——把预留/死代码当作主路径能力写入知识层（EK-19 DOWNGRADED、KO-03 收紧）。此类错误会在所有含模板/配置/预留扩展的仓库反复发生。
- **proposed_change**: Truth Auditor 增加"符号消费验证"机械化检查（方法/模板/配置声明 → 必须有调用者或标注 dead/legacy）；Flow Auditor 每条 Edge 要求可回溯到调用者链。
- **expected_improvement**: 消灭"模板存在即能力"类 False Acceptance。
- **regression_risk**: 低-中（可能误报遗留 API 为 dead——需"预留扩展"豁免规则）

### COV-01 [Coverage Failure · P0]
- **failure_id**: COV-01 ｜ **type**: Coverage Failure ｜ **project**: memgraphrag ｜ **run_id**: ARCH-2026-09-06-001
- **observed_behavior**: `eval(fact_content)` 在 rerank_facts 两处（LLM 生成内容 → eval 解析）——原考古第一阶段未发现，独立审计才定位。安全边界类发现依赖审计运气。
- **evidence**: MemGraphRAG.py rerank_facts 两处 eval(fact_row_dict[...]['content'])
- **root_cause**: coverage-auditor 无"危险动态执行点"扫描清单（skillfortify P-015 已补"失效路径核对"，但未含 eval/exec/subprocess/shell 注入面扫描）
- **impact**: Critical Coverage 下降——安全边界类机制（尤其 LLM 管道项目）系统性漏检。
- **proposed_change**: Coverage Auditor 强制清单增加"动态执行点扫描"（eval/exec/compile/subprocess/shell=True/os.system）并核对输入来源。
- **expected_improvement**: 安全边界发现从"运气"变"必查"。
- **regression_risk**: 低（纯清单增补）

### COV-02 [Coverage Failure · P1]
- **failure_id**: COV-02 ｜ **type**: Coverage Failure ｜ **project**: memgraphrag ｜ **run_id**: ARCH-2026-09-06-001
- **observed_behavior**: `max_qa_steps` 配置定义了（默认 3）但核心类 0 引用——dead config；设计-实现差距检查未覆盖"配置 vs 消费"。
- **evidence**: config_utils.py L241（max_qa_steps）；MemGraphRAG.py 引用 0；retrieval_dataset_test.py L286（传入但未消费）
- **root_cause**: P-014（文档-实现对账）只覆盖"文档声称 vs 实现事实"，未覆盖"配置字段 vs 代码引用"维度
- **impact**: 设计-实现差距检查不完整（配置面漏检）。
- **proposed_change**: Coverage Auditor 增加"配置消费检查"——配置字段 ↔ 代码引用比对，未消费项标 dead-config 或预留。
- **expected_improvement**: 补全 P-014 的配置面。
- **regression_risk**: 低

### FLOW-01 [Flow Failure · P1]（与 EVID-01 同根）
- **failure_id**: FLOW-01 ｜ **type**: Flow Failure ｜ **project**: memgraphrag ｜ **run_id**: ARCH-2026-09-06-001
- **observed_behavior**: F1 控制流 QA 段基于模板推断（"IRCoT 迭代"）而非调用者回溯——Flow Edge 不可回溯到真实调用链。
- **evidence**: 同 EVID-01
- **root_cause**: flow-miner 的 Edge 构建允许"模块存在"推断，未强制"调用者链"证据
- **impact**: Flow Atlas 中死代码路径被画成主路径。
- **proposed_change**: Flow Auditor 强制"每条 Edge 至少一个真实调用者符号"。
- **expected_improvement**: Flow Atlas 只含真实运行时路径。
- **regression_risk**: 低-中

### 其余类型（Discovery/Synthesis/Abstraction/Validation/Orchestration）
- **Discovery Failure**: 无（项目发现正常）
- **Synthesis Failure**: 无重大（Reconciliation 已修正）
- **Abstraction Failure**: 无（KO-03 收紧即可，未过度升维）
- **Validation Failure**: 部分——VAL-01：独立审计靠人工，EVID-01 类错误可由机械化调用图检查自动化（见 P-016）
- **Orchestration Failure**: 无

## Proposals（P0/P1/P2）

| ID | 级别 | 内容 | 风险 | 自动化 Regression |
|----|------|------|------|-------------------|
| P-016 | P0 | 调用图验证（符号消费检查：方法/模板/配置声明必须有调用者或标注 dead） | High（agents/truth-auditor.md + flow-auditor.md） | ✅ R-01 dead-config 消费检查 + 调用者回溯 |
| P-017 | P0 | 动态执行点扫描（eval/exec/subprocess/shell 注入面清单） | High（agents/coverage-auditor.md） | ✅ R-02 eval/动态执行点扫描 |
| P-018 | P1 | 配置消费检查（配置字段 ↔ 代码引用，dead-config 标记） | Medium（references/ 或 agents/） | ✅ 同上 R-01 |
| P-019 | P2 | 数据结构声明 vs 实现回归 fixture（C-08 ThreeLayerMemory 实测模式） | Low（benchmarks/） | ✅ C-08 |

## 治理结论

- P-016/P-017 改动 agents/** → **High：永不自动合并，等待 Owner 人工审查**
- P-018 改动 agents/references → **Medium：需人工审**
- P-019 → Low（可选，本次仅记录）
- 故：创建 Candidate Branch + PR，CI 全 PASS 后仍 **不自动合并**，保留失败/待审证据。
