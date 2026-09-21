# 03 · Knowledge Layer（Generalized KO，L3-L5）— browser-use

> 每个 KO 声明 aggregation_rule（R1 机制簇 / R2 因果链簇 / R3 不变量簇 / R4 主题簇）+ 簇内 EK + 解释范围扩大论证。
> 反退化铁律：禁止"同子系统=聚合理由"。

## KO-01 受控 Agent 循环骨架（R2 因果链簇）
- **aggregation_rule**: R2（因果链簇）——EK-01→EK-02→EK-05→EK-07 沿 causal/mechanism 边形成完整链：骨架→执行防护→动作原子化→失败计数
- **内容**: Web Agent 主循环 = 受控骨架（prepare→decide→execute→post→finalize）+ 每层都有防护（stale-DOM 双层防护、失败计数、停止检查、finally 收尾）。LLM 是循环里的"决策者"但不是"控制者"
- **解释范围扩大**: 从"browser-use 的循环"到"任何 agent 运行时循环"（工具调用 agent / 代码 agent / 终端 agent）——骨架层与决策层解耦是通用形态
- **回溯**: EK-01, EK-02, EK-05, EK-07
- **层**: L3（Pattern，browser-use 验证；同类对照：DeepSeek Harness/OpenClaw 同构——已考古 corpus）

## KO-02 上下文预算分级降级（R2 因果链簇）
- **aggregation_rule**: R2——EK-06（观察端压缩）→EK-04（记忆端 compaction）→EK-04（75% 警告）→EK-04/07（强制 done）形成完整预算链
- **内容**: agent 的上下文/步骤是有限资源，成熟设计按"压缩→警告→强制收尾"分级降级：观察端先压缩（DOM 只留交互元素），记忆端再压缩（compaction），预算 75% 给提示，最后一步/失败强制 done 保全部分结果
- **解释范围扩大**: 任何长时运行 agent（评测/爬取/多步任务）都要回答"资源耗尽时怎么办"——部分结果 > 无结果
- **回溯**: EK-06, EK-04, EK-07
- **层**: L3（Pattern）

## KO-03 软门控：用上下文提示而非硬阻断约束 LLM（R1 机制簇）
- **aggregation_rule**: R1——EK-03（循环检测软 nudge）与 EK-04（预算警告也是 nudge 而非硬停）共享 mechanism 边（"注入上下文消息"）
- **内容**: 对 LLM 的约束分两级：硬门控（强制 done/超时/阻断导航）与软门控（注入上下文 nudge：循环检测提示、预算警告、replan 提示）。browser-use 对"行为质量"（循环/停滞）用软门控——因为 LLM 不能被可靠地硬约束行为，但可以被提示说服；对"资源终止"（预算/失败）用硬门控
- **解释范围扩大**: "哪些约束必须硬、哪些可以软"是 agent 治理的通用问题；可对照 agent-governance-toolkit（确定性门控）与 OpenClaw（sandbox deny）
- **回溯**: EK-03, EK-04, EK-01
- **层**: L3（Pattern，browser-use 强证据：docstring 明示 "never blocks actions"）

## KO-04 事件驱动监视器架构（R4 主题簇）
- **aggregation_rule**: R4——EK-14（EventBus）与 EK-08/09/10/11（安全/权限/captcha/下载监视器）覆盖"浏览器 agent 基础设施"主题的互补维度
- **内容**: 浏览器 agent 的横切关注点（安全/下载/验证码/权限/DOM/崩溃/录制）用事件驱动 watchdog 独立成层，主循环只订阅结果。每个监视器 = 单一关注点的"守护进程"
- **解释范围扩大**: agent 运行时的横切能力（安全、可观测、资源管理）应事件化/插件化，避免主循环膨胀——对照 OpenClaw plugin 架构
- **回溯**: EK-14, EK-08, EK-09, EK-10, EK-11
- **层**: L3（Pattern）

## KO-05 浏览器 agent 的三重安全边界（R3 不变量簇）
- **aggregation_rule**: R3——EK-08（域白名单）、EK-09（最小权限）、EK-10/18（captcha/云）汇聚到同一不变量："agent 只在其被授权的网络与权限范围内行动"
- **内容**: 浏览器 agent 安全 = 域白名单（导航前/重定向后/新标签三层检查）+ 最小权限授予（grantPermissions/cookie 白名单）+ 行为监视（watchdog）。重定向捕获是"先允许后跳转"绕过的关键防御
- **解释范围扩大**: 任何可自主导航的 agent（浏览器/网络爬虫/终端）都需要"授权边界 + 绕过检测"
- **回溯**: EK-08, EK-09, EK-10, EK-18
- **层**: L3（Pattern）

## KO-06 DOM 上下文压缩发生在观察端（R1 机制簇）
- **aggregation_rule**: R1——EK-06（DOM 只序列化交互元素）与 EK-12（LLM 结构化输出）共享 mechanism 边（"压缩/归一化模型输入"）
- **内容**: web agent 的上下文效率瓶颈在观察端：完整 DOM 不可直接喂给 LLM，需要压缩为"交互元素 + selector 索引"；压缩策略 = 过滤非交互 + 缓存复用 + shadow DOM 保留
- **解释范围扩大**: 所有"环境观察型"agent（browser/IDE/桌面）都面临传感器数据 > 上下文的问题——观察端压缩是第一步
- **回溯**: EK-06, EK-12
- **层**: L3（Pattern）

## KO-07 认知模型：软约束与硬约束的分工（L4）
- **aggregation_rule**: R4——KO-03（软门控）+ KO-02（硬预算）抽象为稳定关系
- **内容**: **行为质量靠说服（软），资源终止靠强制（硬）**——LLM agent 中，无法可靠硬控的行为（循环、低效）用上下文提示引导；必须保证的终止（预算、失败）用不可绕过的门控
- **解释范围扩大**: 这是 agent 治理的稳定关系，可解释为什么"指令遵守"不可靠而"工具白名单"可靠
- **回溯**: EK-03, EK-04, EK-07
- **层**: L4（Cognitive Model，browser-use 强证据；跨项目验证 → Candidate C-01）

## KO-08 认知模型：观察-决策-执行-验证的传感器循环（L4）
- **aggregation_rule**: R2——EK-01（循环）+ EK-05（ActionResult 传感器读数）+ EK-03（循环检测）
- **内容**: **Agent 循环是传感器闭环**：观察（DOM 摘要）→ 决策（LLM）→ 执行（动作）→ 验证（ActionResult + watchdog + judge）；LLM 的每一步决策都依赖上一步的结构化"读数"
- **解释范围扩大**: 结构化反馈（ActionResult）质量决定 agent 能力上限——"动作结果即传感器"
- **回溯**: EK-01, EK-05, EK-03
- **层**: L4（Cognitive Model）

## KO-09 认知模型：LLM 的软约束可说服性（L4，保守标注）
- **aggregation_rule**: R1——EK-03 单独强证据 + EK-04 弱化；本 KO 是 KO-03 的抽象深化
- **内容**: **对 LLM 而言，提示是弱约束（可被忽略），API 门控是强约束（不可绕过）；成熟 agent 系统按此分工**（browser-use：循环 nudge 可忽略、max_failures 不可忽略）
- **解释范围扩大**: 设计 agent 治理时先问"这条约束能被忽略吗"——能则考虑软（nudge），不能则硬（门控/白名单/沙箱）
- **回溯**: EK-03, EK-04
- **层**: L4，**Cross-project validation pending**（单项目强证据，跨项目推广未验证；Reconciliation 按独立 Auditor 建议补标注）

## KO-10 方法论：构建 web agent 的五步法（L5）
- **aggregation_rule**: R4——综合 KO-01~06 的可操作化
- **内容**: 构建浏览器 agent：①观察端压缩（只喂交互元素 + 索引）；②动作原子化 + 结构化 ActionResult；③双层 stale-DOM 防护（静态标志 + 运行时检测）；④预算分级降级（压缩→警告→强制 done）；⑤独立验证层（judge + watchdog + 循环检测）
- **解释范围扩大**: 从 browser-use 提炼的可操作构建清单
- **回溯**: EK-01, EK-02, EK-04, EK-05, EK-06, EK-14
- **层**: L5（Methodology）

---

## KO 聚合规则矩阵
| KO | 规则 | 簇内 EK | 边类型 | 解释范围扩大 |
|---|---|---|---|---|
| KO-01 | R2 | EK-01/02/05/07 | causal+mechanism | agent 运行时骨架 |
| KO-02 | R2 | EK-06/04/07 | causal | 资源耗尽策略 |
| KO-03 | R1 | EK-03/04 | mechanism | 约束哲学 |
| KO-04 | R4 | EK-14/08/09/10/11 | subsystem | 横切能力分层 |
| KO-05 | R3 | EK-08/09/10/18 | constraint | 授权边界 |
| KO-06 | R1 | EK-06/12 | mechanism | 观察端压缩 |
| KO-07 | R4 | EK-03/04/07 | —（抽象） | agent 治理 |
| KO-08 | R2 | EK-01/05/03 | causal | 传感器循环 |
| KO-09 | R1 | EK-03/04 | mechanism | 软约束可说服性 |
| KO-10 | R4 | EK-01/02/04/05/06/14 | —（综合） | 构建清单 |

## 配比检查
- KO 总数：10（7-12 目标区间内）
- L3 Pattern：6（KO-01~06）｜ L4 Cognitive Model：3（KO-07/08/09）｜ L5 Methodology：1（KO-10）
- 全部 KO 可回溯 EK（无悬空升维）；无"同子系统=聚合理由"的假聚合
