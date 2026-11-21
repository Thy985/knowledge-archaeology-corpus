# 00 · Overview — RAMPART Knowledge Archaeology（refresh, ARCH-2026-10-18-001）

## 一句话定位
RAMPART 是一个 **pytest-native 的 Agentic AI 安全/安全测试框架**（Microsoft AI Red Team），把攻击/探针/评测/报告全流程嵌入 pytest 生态。本次为 **refresh 考古**（基线 ARCH-2026-09-04-001 @ 125595c）。

## 为什么 refresh 这个项目
09-04 考古后 6 周内，RAMPART 合入 5 个 feature 级 PR + 2 个修复 PR，10-15 仍持续 push；核心增量集中在**评测证据契约化**（trace runner / versioned serializer / compatibility gate / trial population / onboarding skill）——这是一条清晰的"把评估从隐式约定演进为显式契约"的主线，与已考古 PyRIT 同生态，并直接衔接 10-19 起的 Harness/治理考古领域。

## 本次 refresh 交付了什么
| 层 | refresh 增量 |
|---|---|
| Project Layer | 架构 delta：trace/serialization/_schema/_population/_trial/skills 6 个新模块 + schemas/ 目录 |
| Engineering Knowledge | **15 条新 EK（EK-R01~R17，含 Reconciliation 补充 2 条）**，全部带 links（EK Graph） |
| Knowledge Layer | **5 个新 KO（KO-R01~R05）**，聚合规则 100% 覆盖（R1×4 / R2×1），L3×2 + L4×3 |
| Flow Atlas | 7 类流 refresh 增量（Control 线性 trace / State 终局字段 / Data 序列化管道 / Evidence 分层 / Authority xdist / Memory 复用 / Policy 契约闭环） |
| Candidates | 6 条（C-R01~R06）+ Reconciliation 追加 C-R07 |
| Validation | 主考古六审 PASS + **独立 Auditor 盲重建**：CONFIRMED 13 / PARTIALLY_CONFIRMED 1 / MISSING 3 / 其余 0 |

## 核心认知增量（一句话版）
1. **终局证据与执行证据分离**（KO-R01）：stop-check 在线评估 ≠ final-trace 终局判定，codec 永不推断
2. **契约兼容性 = 显式决策**（KO-R02）：指纹→决策→门禁闭环，禁止 relabel/upcast
3. **判定者与被判定历史隔离**（KO-R03）：raw_turns 永不带框架标注
4. **schema 策略与 live 类型分离**（KO-R04）：序列化严格不污染运行时宽松
5. **参数类型纪律**（KO-R05）：显式拒绝 bool/非有限值

## 关键文件
- 01_project-map：架构 delta + 数据结构 delta
- 02_ek_graph：EK-R01~R17（含 EK Graph 子图 + 与 09-04 基线 5 条连接）
- 03_ko：KO-R01~R05（aggregation_rule 矩阵）
- 04_flow_atlas：7 类流 refresh 增量（Edge 可回溯 symbol/file/condition）
- 05_candidates：C-R01~R07（Hypothesis/Observation 严格标注）
- 06_validation：六审 + 质量指标
- 07-independent-validation：Auditor 盲重建报告（14 Findings + 判定统计）
- run_metadata.yaml：run 级元数据
- reconciliation.md：阶段⑤修正合并记录

## 事实来源
全部事实可回溯 HEAD `0fb9cc2ff8da8cbba0537444b3f2313cf86e032f`（main，2026-10-15）仓库实际内容（文件:行）。基线差异对照 `git log 125595c..HEAD`。
