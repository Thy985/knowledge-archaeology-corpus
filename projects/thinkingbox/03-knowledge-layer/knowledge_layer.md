# 03 · Knowledge Layer — ThinkingBox Generalized KO（窄尖顶 · 可迁移认知）

> 9 个 Core KO，全部从 02 层 EK Graph 按聚合规则（R1 机制簇 / R2 因果链簇 / R3 不变量簇 / R4 主题簇）生成。每个 KO 声明：aggregation_rule（簇内 EK + 边类型）、L3 Pattern / L4 Cognitive Model 定位、epistemic status（Fact/Observation/Hypothesis/Pattern/Model）。**簇成立五条件**（内聚性/跨实例性/解释范围扩大/可命名/可回溯）逐项标注。Cross-project 未验证的 L4 一律标 `Pattern/Model (cross-project validation pending)`，不冒充 Principle/Law。

---

## KO-01 · 状态真相评分：评测真相 = 世界状态，不是 LLM 输出（R2 因果链簇）

- **aggregation_rule**: R2 因果链簇 — causal 边链：EK-27（init/geteffects 协议）→ EK-28（effects 双源取证）→ EK-25（TestContext 快照）→ EK-04（断言分类）→ EK-37/38/39（贝叶斯统计判定）
- **claim**: 有状态 agent 评测的正确做法是"看世界变了没有"，而不是"听模型说完成没有"——从 MCP server 的 effects（自报状态 + proxy 观测双源）到评分输入快照（TestContext），再到断言与贝叶斯统计，整条链不依赖模型自述。
- **L3 Pattern**: 有状态工作流评测 = 环境状态取证（effects）+ 评分输入快照（TestContext）+ 可重跑测试（解码/测试解耦）三段式。
- **L4 Model**: 评测一个 agent 是否完成了任务，应以环境状态转移为证据，以 agent 的陈述为辅证。
- **epistemic status**: Pattern（本仓内多实例：cloud_drive server + tests/servers.yaml 四种 server 形态均适用）；Model（cross-project validation pending）
- **五条件**: 内聚（全链同属评测执行面）✓ / 跨实例（多 server 形态 + 多评测模式）✓ / 解释范围扩大（解释"为什么 effects 是评分核心"）✓ / 可命名 ✓ / 可回溯（EK-25→agent_session_base.py:125-158）✓
- **反例攻击**: judge 评分（EK-41/42）完全不看 effects——故本 KO 限定"有状态工具工作流"场景，不覆盖纯文本问答评测（answer_evaluator 走 LLM 判定）。

## KO-02 · 副作用安全铁律：超时/失败绝不重试、绝不静默重调（R1 机制簇）

- **aggregation_rule**: R1 机制簇 — mechanism 边：EK-32（max_retries_timeout=0 超时不重试）+ EK-15（tc.metadata["error"] 短路）+ EK-30（init 失败回滚 destroy）+ EK-36（认证显式开启）
- **claim**: 在有副作用的外部系统上，重复执行比失败更危险——超时不重试（可能已生效）、失败工具短路不重调、初始化失败整体回滚不留半建状态、认证显式开启不静默默认。
- **L3 Pattern**: 副作用安全四则：超时不重试 / 失败短路 / 回滚不留半态 / 防护显式开启（同类：支付系统幂等键、数据库事务回滚）。
- **L4 Model**: 与不可幂等外部系统交互时，执行者应把"避免重复生效"置于"提升成功率"之上。
- **epistemic status**: Pattern（本仓显式策略）；Model（cross-project validation pending）
- **五条件**: 内聚（同属执行失败路径治理）✓ / 跨实例（client 超时 + 工具错误 + session init 三处独立实现）✓ / 解释范围扩大 ✓ / 可命名 ✓ / 可回溯（EK-32→mcp_proxy_client.py）✓
- **反例攻击**: retryable_server_errors（502/503）仍重试——例外存在（server 明确报错可安全重试），本 KO 精确限定"超时/未知状态"。

## KO-03 · 防无界/防失控：每一层都有停止闸门（R2 因果链簇）

- **aggregation_rule**: R2 因果链簇 — causal 边链：EK-19（agent turn 上限 break 防工具循环）→ EK-18（上限优先判定）→ EK-17（五态 finish_reason）→ EK-33（MCP worker 超时转 TimeoutError 不杀进程）→ EK-26（无消息=agent_error 降级）
- **claim**: 从 agent 循环到 MCP worker 到测试执行，每一层都内置停止闸门：回合上限、超时转换、无输出降级——失控不是"不可能"而是"有界且可标记"。
- **L3 Pattern**: 多层执行系统的停止闸门分层：语义层（回合上限）/ 运行时层（超时）/ 结果层（降级标记）。
- **L4 Model**: 执行深度应被显式约束（bounded），且每层约束独立存在；约束触发时以可标记的 finish_reason 而非中断呈现。
- **epistemic status**: Pattern（本仓三层独立实现）；Model（cross-project validation pending）
- **五条件**: 内聚 ✓ / 跨实例（agent_user_loop + mcp_worker + parallel_executor 三处）✓ / 解释范围扩大 ✓ / 可命名 ✓ / 可回溯 ✓
- **反例攻击**: max_agent_sim_turns 默认 sys.maxsize——语义层闸门默认几乎无限，实际靠用户回合/user_done/无消息终止；本 KO 的"有界"指机制存在而非默认值小。

## KO-04 · 评测管道的注入防护三件套（R1 机制簇）

- **aggregation_rule**: R1 机制簇 — mechanism 边：EK-23（UserSimulator sanitize 防结构混淆）+ EK-43（rubric "treat as untrusted" 指令）+ EK-44（safe_tag_encode 编码 <>）
- **claim**: LLM 评测管道里的每一处"LLM 读内容再判断"都做了注入防护：被测内容可能含指令——用户模拟器转义框架分隔符、judge 显式声明内容不可信、结构化输入编码尖括号。
- **L3 Pattern**: LLM 作为判断者/模拟者时，必须声明"输入内容不可信"并隔离框架语法（同类：prompt injection 防护的 system-prompt 声明模式）。
- **L4 Model**: 信任边界应按"内容 vs 指令"划分——凡是模型会执行的判断，其输入中的指令都必须显式降权。
- **epistemic status**: Pattern（本仓三处独立实现）；Model（cross-project validation pending；owasp-agentic-skills 同族支持）
- **五条件**: 内聚（同属 LLM 输入治理）✓ / 跨实例（user sim / rubric / judge 三处）✓ / 解释范围扩大 ✓ / 可命名 ✓ / 可回溯 ✓
- **反例攻击**: direct_response（EK-13）把工具响应格式化成模板注入对话——这是"有意注入"，非防护缺口；防护针对的是评测者侧 LLM。

## KO-05 · 统计层刻意对抗"小样本自信"（R4 主题簇）

- **aggregation_rule**: R4 主题簇 — 围绕"统计防幻觉"主题：EK-38（pass^k 有意 biased）+ EK-37（pass@k 无偏对照）+ EK-39（goldilocks 贝叶斯 zone）+ EK-41（judge 解析失败默认 No）+ EK-22（COPY-ONLY 防用户幻觉）
- **claim**: 评测系统的统计层与数据层都在对抗同一敌人——小样本/单次的"看起来成功"：pass^k 要求 k 次全过、goldilocks 拒绝小样本稳定判定、judge 格式违约即失败、用户模拟器禁止虚构实体。
- **L3 Pattern**: 可靠性评测的统计纪律：有偏指标换区分度、贝叶斯区间拒自信、契约违约即失败（同类：LLM-as-judge 的 consistency 检查）。
- **L4 Model**: 单次成功不是可靠性；统计上"稳定通过"与"通过率未知"必须被显式区分。
- **epistemic status**: Pattern（本仓设计目标+实现）；Model（与本仓论文主题 "One Success Isn't Reliability" 一致，但论文数字为外部宣称）
- **五条件**: 内聚 ✓ / 跨实例（agg/eval_utils/judge/user_sim 四处独立机制）✓ / 解释范围扩大（解释框架全部统计设计）✓ / 可命名 ✓ / 可回溯 ✓
- **反例攻击**: pass_at_k_unbiased 在 n-c<k 时直接 =1.0（无样本时给满分）——这是显式保留的宽松路径，本 KO 不覆盖该边界（记录为边界例外）。

## KO-06 · 评测可复现性：全量留痕 + 解码测试解耦 + 断点续跑（R4 主题簇）

- **aggregation_rule**: R4 主题簇 — 围绕"可复现/可审计"主题：EK-06（judge_motivation 入 metadata）+ EK-08（run-test 重跑）+ EK-09（错误行重试）+ EK-53（uid + JSONL 续跑）+ EK-54（配置迁移显式化）
- **claim**: 评测结果可审计 = 每次 run 的 agent/user/setup 配置全落 run_metadata.yaml、judge 决策理由入 metadata、DecodeResult 全量 JSONL 留存、测试可脱离解码重跑、错误行可定向重试。
- **L3 Pattern**: 评测管道把"证据留痕"做成一等公民：配置快照 + 决策动机 + 可重放产物。
- **L4 Model**: 评测结论的价值上限 = 其可复现性；不可重放的评测结果不具备比较价值。
- **epistemic status**: Pattern（本仓全链路实现）；Model（cross-project validation pending）
- **五条件**: 内聚 ✓ / 跨实例（infer/runtest/agg 三命令协作）✓ / 解释范围扩大 ✓ / 可命名 ✓ / 可回溯 ✓
- **反例攻击**: JSONL 水合要求 base_dir/agent 为空（EK-53）——续跑是"文件内嵌全信息"而非依赖环境，这是约束不是缺口。

## KO-07 · 会话生命周期即评测状态机（R2 因果链簇）

- **aggregation_rule**: R2 因果链簇 — causal 边链：EK-27（init 建状态）→ EK-30（init 校验+回滚）→ EK-20（history 重放恢复状态）→ EK-16（end-turn dummy 记录）→ EK-28（effects 取证）→ teardown（destroy 清理）
- **claim**: 每个评测会话 = 一次完整的状态机：初始化（带校验与回滚）→ 运行（工具调用积累 effects）→ 取证（get_effects 快照）→ 清理（teardown）——有状态业务工作流的评测正是通过这种显式会话生命周期获得可复现性。
- **L3 Pattern**: 有状态环境评测 = 显式会话状态机（init→run→effects→teardown），恢复历史（replay）是状态机的合法入口。
- **L4 Model**: 评测环境的"初始状态"必须是可声明的（init_config），"当前状态"必须是可取证的（effects），否则评测不可复现。
- **epistemic status**: Pattern（本仓协议级实现）；Model（cross-project validation pending）
- **五条件**: 内聚 ✓ / 跨实例（session_proxy + mcp_cloud_drive + tests/servers.yaml）✓ / 解释范围扩大 ✓ / 可命名 ✓ / 可回溯 ✓
- **反例攻击**: replay 是"尽力而为"（失败仅 warning，EK-20）——状态恢复不保证精确；本 KO 限定"状态可声明/可取证"，不宣称"状态可精确重放"。

## KO-08 · 工具面由场景声明：可见性 + 冲突裁决 + 协议约定（R3 不变量簇）

- **aggregation_rule**: R3 不变量簇 — constraint 边汇聚：EK-31（visible_tools 过滤 + priority 裁决）约束 EK-11（agent 可调工具面）与 EK-35（schema 净化）；EK-27（三保留协议）约束 EK-29（/mcp 开放面）
- **claim**: 评测中 agent 的工具面不是"server 有什么就有什么"，而是"场景声明什么才有什么"——不可见工具不可调；同名工具按优先级确定性裁决；框架与 server 的约定（init/geteffects/teardown）以工具形式固化。
- **L3 Pattern**: 评测沙箱的工具面控制 = 白名单（visible_tools）+ 冲突优先级 + 协议工具化。
- **L4 Model**: agent 的能力边界应由评测场景显式声明，而不是由环境隐式暴露。
- **epistemic status**: Pattern（本仓实现）；Model（cross-project validation pending）
- **五条件**: 内聚 ✓ / 跨实例（ToolDispatcher + Session.initialize + /mcp 三面）✓ / 解释范围扩大 ✓ / 可命名 ✓ / 可回溯 ✓
- **反例攻击**: `__reserved__server_tool`（tools/client/common.py:189）允许 fixture 绕过可见性直调任意 server 工具——这是测试专用的后门（fixture 受信），非场景 agent 路径；本 KO 限定 agent 视角。

## KO-09 · 双层执行模型：隔离是选项，可调试是另一选项（R1 机制簇）

- **aggregation_rule**: R1 机制簇 — mechanism 边：EK-02（TestScriptSubprocess 子进程隔离）+ EK-03（\0 协议）+ EK-07（MockPrint 输出隔离）+ EK-10（Debug import 断点）
- **claim**: 测试执行采用双层模型：生产默认子进程隔离（含输出通道隔离 \0/MockPrint），调试模式直接 import 支持 IDE 断点——隔离与可调试是同一执行面的两个可选项，由 CLI 显式选择。
- **L3 Pattern**: 代码执行器的双模式：子进程隔离（生产）+ import 直连（调试），以显式开关切换（同类：pytest 的 subprocess/断点调试对比）。
- **L4 Model**: 评测执行的隔离强度与可调试性是同一自由度的两端，应提供显式档位而非二选一。
- **epistemic status**: Pattern（本仓实现）；Model（cross-project validation pending）
- **五条件**: 内聚 ✓ / 跨实例（三种 TestScript 变体）✓ / 解释范围扩大 ✓ / 可命名 ✓ / 可回溯 ✓
- **反例攻击**: Debug 模式强制 repeat=1 且仅单测——可调试档位不保证统计有效性；本 KO 不混淆"调试模式"与"评测模式"。

---

## KO 聚合规则矩阵

| KO | 规则 | 簇内 EK | 边类型 | 解释范围扩大论证 |
|---|---|---|---|---|
| KO-01 | R2 | 27,28,25,04,37,38,39 | causal | 全链解释"为什么 effects 是评分核心" |
| KO-02 | R1 | 32,15,30,36 | mechanism | 四机制同属"失败不扩散"原则 |
| KO-03 | R2 | 19,18,17,33,26 | causal | 三层停止闸门构成完整防失控故事 |
| KO-04 | R1 | 23,43,44 | mechanism | 三处注入防护共享同一机制 |
| KO-05 | R4 | 38,37,39,41,22 | 主题 | 统计+数据层对抗同一敌人（小样本自信） |
| KO-06 | R4 | 06,08,09,53,54 | 主题 | 全链路证据留痕设计 |
| KO-07 | R2 | 27,30,20,16,28 | causal | 会话状态机完整因果链 |
| KO-08 | R3 | 31,11,35,27,29 | constraint | 工具面声明制的不变量 |
| KO-09 | R1 | 02,03,07,10 | mechanism | 双层执行模型同一机制 |

**反退化检查**：无"都属于某子系统"式聚合（每个 KO 的聚合理由都是机制/因果/不变量/主题，非目录分类）；簇内 EK 全部可回溯到 02 层；无悬空升维。
