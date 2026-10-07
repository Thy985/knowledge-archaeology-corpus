# 06 Validation & Evidence — microsoft/Webwright

## 6.0 Validation 摘要
| 维度 | 结论 |
|---|---|
| Truth | 44/44 EK 直接溯源源码/README/测试；0 条无源 |
| Coverage | 覆盖 agents/environments/models/run/tools/skill_factory/config/tests/CI/plugin/examples 全子系统 |
| Flow | 七类流全部符号可回溯（file:symbol:condition） |
| Abstraction | 8 KO 全部 R1-R4 聚合；无悬空升维；无"同子系统=聚合"假聚合 |
| Counterexample | 定向反例 4 组（见 6.4），未弱化标准 |
| Epistemic | Fact/Observation/Hypothesis 严格分离；基准数字标 Observation（未实测） |

## 6.1 证据强度（Evidence Strength）
| 级 | 数量 | 说明 |
|---|---|---|
| S3 已实现 | 38 | 机制/实现/决策，直接读源码确认 |
| S4 测试验证 | 5 | gate/route/recommend/evolve/tool_routing（LLM-free 测试揭示） |
| S5 运行验证 | 0 | 本考古未运行代码（无模型 key/浏览器） |
| S7/S8 | 0 | 未人工实测/跨项目未验证 |

## 6.2 Blind Reconstruction（盲重建，阶段⑤独立 Auditor 详见 independent-audit/report.md）
Auditor 不引用本包，独立重读仓库关键声明后对比。结果摘要：
- CONFIRMED ×3：Self-Reflection Gate 硬逻辑（default.py:202-268）；Skill Factory route 三决策 + fallback 语义（route.py + test_route.py）；done 降级防御（mb.py:107-121）。
- CONTRADICTED ×0 / DOWNGRADED ×1 / OVER_GENERALIZED ×0。
- PARTIALLY_CONFIRMED ×2（基准数字不可复现；插件模式门控未知）。
- MISSING ×2（retrieve 细节、docs/skill_factory 两份文档未精读——补充见 Reconciliation）。

## 6.3 反例攻击（Counterexample）
| # | 目标 | 反例 | 处理 |
|---|---|---|---|
| CE-1 | KO-01（验证门） | local_browser 模式 require_self_reflection_success=false，模型自裁 | KO-01 适用域显式限定 workspace 模式；KO-08 承接 |
| CE-2 | KO-04（零 token 复用） | `_memorized_answer`：脚本含答案 verbatim 的 solve 被丢弃，无技能产生 | KO-04 补充"蒸馏的是方法不是答案" |
| CE-3 | KO-06（形状不变量） | parse 层 demote done 是"容忍"而非严格拒绝 | KO-06 注明平衡点：降级 vs 拒绝并存 |
| CE-4 | EK-38（learn 管道） | self_verify 无 gold 时 agent 误信的答案仍 PASS（learn.py 自证警告） | 保留为 gate 弱门事实 + C-02 候选 |

## 6.4 Contradictions / 边界记录
- 无代码级矛盾（README 声明 vs 实现一致：规模、模式、无多 agent 声明属实——仓库无 agent 编排框架依赖，仅 asyncio/typer 等）。
- "~1.5k LoC 核心"与实测 8061 行（含 skill_factory）的差异：README 数字为 Skill Factory 合并前或仅核心 loop；本考古以实测 8061 行为准（快照文档记录）。

## 6.5 质量指标（写入 run_metadata.yaml）
facts 31 / EK 44（links 100%、游离 0）/ KO 8（聚合 100%）/ Candidates 6 / flow_edges_traceable true / validation 判定统计见 independent-audit。

## 6.6 Reconciliation（与阶段⑤合并的修正）
1. **管道顺序**：确认 step_limit 检查在 query 前（default.py:386-392），修正在 Flow Atlas 顺序描述。
2. **遗漏补充**：retrieve.py `simple` 方法机制、docs/skill_factory/{manual,reference}.md 存在未精读（标入 C-05 注记），不阻塞结论。
3. **Scope 收紧**：KO-01/KO-08 完成判定结论明确限定双模式语境，不泛化为"所有 agent 必须外部门"。
4. 基准数字全部标 Observation（C-05），未升维。
5. **Auditor 修正并入**（independent-audit/report.md）：①EK-34 补 recommend 三重防御（幻觉 id 拦截、库缺失 warning、decide/promote 分层）；②EK-40 补 `_well_shaped` 诚实边界（run 答案仅 shape 保证）；③KO-08 提升为显式对照（门强度绑定状态模型）；④无 contradicted，1 DOWNGRADED 已按上述收敛。

## 6.7 本包自我声明
- 未运行仓库代码/测试（无 key）；静态证据链完整。
- 未修改目标仓库任何文件（只读约束）。
- 不覆盖 corpus 历史：Webwright 为首个 run（去重基准核对通过）。
