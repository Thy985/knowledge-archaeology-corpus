# 00 · Overview — AProver（BMC-Agent）考古总览

> run_id: ARCH-2026-09-28-001 ｜ commit: b9314c704c1142d195ae43abde4ad9e03f84aaa8 ｜ skill: knowledge-archaeology v3.2

## 一句话定位
**AProver / BMC-Agent 是 *agentic model checking*（代理化模型检查）的原型实现**：把"LLM 语义推理 + 健全有界模型检查（CBMC/Kani/JBMC）"配对成一个自动验证管线——agent 生成规格、分类反例、精化规格；solver 在展开界内提供形式保证。核心命题：**"agents propose, conventional tools dispose"**——LLM 的每个健全性相关决策必须经传统检查（CBMC 查询 / SMT 守卫 / 运行时复现）才能影响最终判定。

## 它让我们认识到了什么（核心认知，详细论证见 03/05/06）
1. **删除不对称性（健全性第一定律）**：从报告中静默删除一个真实 bug（假阴性）是健全性损失；错误保留只是噪声（精度损失）。因此"排除一个发现"与"降级一个发现"是**非对称**操作，必须按证据类型授权——`soundness_policy.py` 把这一点固化为可审计的单一事实源。
2. **智能 ≠ 权威（在验证语境）**：LLM 判定（agentic judgment）非确定性、可能自信地错，因此**只能降级证据档位（RE-TIER），永不可作为唯一删除依据**；只有确定性验证器（CBMC 重验）或自验证证人（编译+精确复现）才可 DELETE。
3. **证据分级 = 信任分层**：越接近确定性事实的档位越可信——动态复现（confirmed_dynamic）＞ 调用链回溯（confirmed_system_entry）＞ BMC 输入可达（confirmed_bmc）＞ 过度精化守卫（likely）＞ 真实性否决（unlikely）。realism 审计只能降档不能丢弃（保留完整审计轨迹）。
4. **确定性短路替代 LLM 判定**：realism 检查用 witness-pattern（未初始化库全局 / jq stub 断开 / NULL-guard 违背）做确定性预检，直接返回 UNREALISTIC，省一次 LLM 往返且避免 LLM"创造性地"合理化人为产物——用模式库替代不确定判定。
5. **混合验证的分工准则（L5 方法论）**：语义推理给 LLM（规格生成/反例分类/精化建议），形式保证给 solver（BMC/展开界），删除决策只给确定性证据；每一层用更确定的工具兜底下一层的不确定性。

## 三层知识配比
| 层 | 数量 | 说明 |
|---|---|---|
| Project Layer | 1 地图 | 定位/架构/模块/生命周期/配置/依赖（见 01） |
| Engineering Knowledge（EK Graph） | 21 条（含 links 六类边） | 见 02 |
| Generalized KO | 7 个（R1-R4 聚合规则） | 见 03 |
| Candidates | 5 个 | 见 05 |
| Validation | 独立盲验证 22 条判定 | 见 06 |

## 关键事实锚点（可追溯）
- 设计原则声明：README.md L16-18（"agents propose, conventional tools dispose"）
- 健全性规则单一事实源：`bmc_agent/soundness_policy.py`（96 行，模块 docstring 明言"no real bug is ever silently removed from the report"）
- 管线：README L24-46（Phase 1-4）+ PIPELINE.md（ASCII 全流程）
- 五档证据分级：README L40-48 + PIPELINE.md Tier assignment + bug_reporter.py
- realism 确定性短路：realism_checker.py L150-235（witness-pattern / jv stub-disconnect / NULL-guard violation）
- 实测：README Evaluation 节（VibeOS 13 bugs / llm.c 22-of-30）+ findings/ 目录
- 本机实测：soundness_policy 行为全符合文档；pytest 抽样 213 passed / 1 failed（测试-实现-文档漂移，见 02 EK-16）
