# Reconciliation — deepseek-harness（ARCH-2026-09-03-001）

> 输入：原始 Archaeology Result（00-06 + run_metadata）+ Independent Validation Report（07，独立 Auditor 盲重建）。
> 本文件记录两者冲突的**裁决结果**与**对 Corpus Artifact 的修正**。
> 原 Archaeology Result 未改动（完整保留于 `archaeology-runs/ARCH-2026-09-03-001/`）；修正只体现在本 Reconciled 顶层产物。

## 一、裁决原则
1. Repository is the Source of Truth：与原结果冲突时以仓库实际代码为准。
2. 独立 Auditor 的代码实证优先于原报告的符号推断。
3. 修正 Corpus Artifact，不修改生产 Skill（本 run 不产生 Skill 修改请求）。

## 二、冲突裁决表
| # | 对象 | 原结果 | 独立验证 | 裁决 | 处理 |
|---|------|--------|---------|------|------|
| R-1 | F-05 / EK-07 管道顺序 | approval 在 guards **之后** | approval 在 guards **之前**（`tools/index.ts:1367,1400,1468-1493`：collapse→pre-execute→approval(ask)→guards→execute） | **修正原结果** | 04-flow-atlas F-05 重写 chain；02-ek-graph EK-07 claim 更新 |
| R-2 | KO-03 scope | fail-closed 族覆盖全部权限面 | 反例：plan-mode 等非受限路径不适用 | **收紧 scope** | 03-ko KO-03 claim 限定"受限执行路径（受控工具调用）"，仍 L4 |
| R-3 | 覆盖缺口 M-1..M-6 | 未记录 | approval delegation 播种 / PTC collapse / plan-mode / settings seam / repeat-tool-reminder / legacy 硬拒绝 | **补充** | 新增 EK-27..EK-32，并入对应 KO 簇 |
| R-4 | 覆盖缺口 M-7 | 已登记 web/acp | 扩展：goal/schedule/spill/mcp/feedback/jobs/code-runtime 未深度考古 | **登记** | metadata.coverage_gaps 登记；不强行造 EK |
| R-5 | 反例与矛盾 | 3 反例（C-06/07/08）+ 3 矛盾 | 支持保留 | **保留** | 05-candidates 保留全部 Counterexample / Contradiction |

## 三、保留的 Contradictions（不抹平）
- **compaction 有损**：summary 节点替换历史 = 信息有损压缩；与"日志即真相"构成张力（已登记，未升维）。
- **SEARCH_FAILED 掩盖**：broad 签名可掩盖真实错误，与"结构化错误归属"冲突（已条件化，EK-15/ KO-07）。
- **danger-full-access 绕过**：预置 full-access profile 可绕过渐进式审批，是 fail-closed 的反例（已条件化，C-06）。

## 四、保留的 Counterexamples（C-06..C-08）
见 `05_candidates/candidates.md`——全部作为反例保留，未因"验证通过"而删除。

## 五、Epistemic 纪律（Reconciled 后再次确认）
- Hypothesis 未写成 Fact：EK-14 / EK-22 等标 Observation；C-01..C-09 全部进 Candidates。
- Cross-project Candidate 未写成已验证 Principle：KO-03 / KO-05 均标 `Cross-project validation pending`；无 L5。

## 六、是否产生 Skill Improvement Proposal
**是（建议级，不修改 Skill）**：独立验证提出 5 个可作 skill CI Gold Record 的 Benchmark / Regression case（B-1..B-5）：
- B-1 `approval.spec.ts:430` — never 策略不可绕过（"never unbypassable"）
- B-2 `scoped.spec.ts:303` — 守卫单调性（force allow 不能 bypass guard）
- B-3 `tools/index.ts:1367` — collapse 先于策略管线（确定性失败）
- B-4 plan-mode 恢复（projection 折叠/恢复）
- B-5 legacy 硬拒绝（格式演进边界）

> 注：本 run **不修改** `knowledge-archaeology-skill`；B-1..B-5 作为建议写入 PR，供 Skill 维护者评审。

## 七、验证结论（Reconciled 后）
- Original Validation：ACCEPT（6 Validator 全 PASS + Blind Reconstruction 8/8 无推翻 + Reconciler 4 冲突 CONDITIONAL）
- Independent Validation：CONFIRMED 34 / PARTIALLY_CONFIRMED 6 / INCORRECT 1（F-05）/ MISSING 7 / CONTRADICTED 0
- Reconciled 质量指标：32 EK / 8 KO / 游离 0% / 聚合 100% / 簇规模 5.0
