# 03 · Knowledge Layer（Generalized KO）— browser-use 1.0.8rc1

> Layer mapping: L1 Fact (raw evidence) → L2 Observation/Engineering Knowledge (EK) → L3 Pattern → L4 Cognitive Model → L5 Methodology.
>
> Epistemic discipline: L3 items are **project-validated patterns**, not universal principles. L4 items are **models** and are marked `Hypothesis` where cross-project evidence is absent. L5 is a proposed methodology, explicitly `Cross-project validation pending`.
>
> Each KO declares an aggregation rule and a traceable EK chain. No KO is formed by “same subsystem = cluster”.

## KO-01 执行单元迁移：从动作过滤到代码权威（R1 机制簇）

- **aggregation_rule**: R1（机制簇）——EK-04（Python cell）→ EK-05（串行 cell）→ EK-06（进程组回收）→ EK-07（worker 传输）→ EK-08（env allowlist）→ EK-09（secret 控制通道）共享同一机制：把“模型意图”变成“受控进程中的代码执行”。
- **内容**: 1.0 把执行单元从“每步一个 DOM 动作”换成“持久 Python REPL 中的一个 cell”。模型获得的是代码执行能力，而不是固定动作集合；运行时的控制点随之下移为进程启动、环境注入、cell 串行、进程组击杀、返回通道 redaction。
- **解释范围扩大**: 任何“让模型直接操作环境”的 agent（浏览器、终端、IDE、桌面）都要回答同一个问题：授权是给“动作”还是给“代码”？给动作可以枚举白名单；给代码只能靠进程/环境/通道/资源边界。
- **边界与反例**: 这不是“代码执行一定更危险”或“动作白名单一定更安全”。本项目里，代码路径反而有独立 worker、env allowlist、secret 注入和 checkpoint redaction；MCP 路径同样是代码执行，却没有这些层（EK-16）。所以结论只在“本项目的 Agent 路径”成立。
- **回溯**: EK-04, EK-05, EK-06, EK-07, EK-08, EK-09；证据 EV-09, EV-27, EV-37 至 EV-52。
- **层**: L3（Pattern，项目内验证）。
- **epistemic_status**: Validated Pattern（仅限本项目 1.0.8rc1 Agent 路径）。

## KO-02 策略按执行点分层，声明强度取决于最弱模式（R3 不变量簇）

- **aggregation_rule**: R3（不变量簇）——EK-12（域名匹配）→ EK-13（Guard）→ EK-14（Guard fail-open）→ EK-15（attached Chrome 缺层）→ EK-17（MCP cloud 拒绝）共同约束同一不变量：“任何被声明的域名限制，只能在该模式实际启用的层上成立”。
- **内容**: domain policy 有三层：helper 预检、浏览器级 Guard 的 main-frame Document 拦截、cell 后 sweep。不同 Browser 模式启用不同层：cloud/chromium/profile-chrome 三层；attached `Browser('chrome')` 只有 layer 1 + layer 3；MCP 只有在自有 Browser 且显式配置 domains 时才有 Guard；cloud MCP 对 admin 调用直接拒绝。
- **解释范围扩大**: 安全/合规声明应按“执行点”拆开，而不是按“产品功能”合并。一个 UI 开关写着“限制域名”，并不等于所有路径都被限制。
- **边界与反例**: Guard 断开期间 fail-open；`Runtime.evaluate` 导航、iframe、子资源、worker 自己的 `requests/socket/curl` 不在拦截范围；`Browser('chrome')` 的用户 tab 被明确豁免。因此“三层”不是“全路径硬阻断”。
- **回溯**: EK-12, EK-13, EK-14, EK-15, EK-17；证据 EV-13, EV-61 至 EV-68, EV-94。
- **层**: L3（Pattern，项目内强证据 + 文档承认的边界）。
- **epistemic_status**: Validated Pattern（限定 browser-use 的四种 Browser 模式与 MCP 路径）。

## KO-03 Secret 是通道级 redaction，不是机器级隔离（R3 不变量簇）

- **aggregation_rule**: R3——EK-09（控制通道注入）→ EK-10（redaction 形态与残余）→ EK-11（域绑定 fail-closed）→ EK-28（key 存储）→ EK-31（telemetry scrub）共同约束：“secret 只在被覆盖的返回/发送通道被替换，未覆盖通道仍是明文能力”。
- **内容**: `sensitive_data` 经 REPL 控制通道进入 worker，返回的 output/error/value/checkpoint/partial 和 telemetry 字段被 redaction/scrub；但 worker 与用户同权限，能读文件、`/proc/<pid>/environ`、`.env`、截图、live view、recording；`open()` 写文件、图像像素、短于 4 字符的 secret 不在覆盖内。
- **解释范围扩大**: 任何“把密码交给 agent”的产品都应区分三种承诺：不进入模型上下文、不进入遥测、不能被同机进程读取。三者不是同一件事；只有前两者可由 redaction 近似实现。
- **边界与反例**: 文档与测试都承认 worker 可访问父进程 key；`type_secret` 的域检查与键盘输入之间存在 focus race；这不是“已隔离”，只是“尽量不把明文送到模型和日志”。
- **回溯**: EK-09, EK-10, EK-11, EK-28, EK-31；证据 EV-14, EV-38, EV-46, EV-48 至 EV-52, EV-86 至 EV-89。
- **层**: L3（Pattern，项目内验证）。
- **epistemic_status**: Validated Pattern（限定本项目 Agent/MCP/telemetry 通道）。

## KO-04 花费与失败前置：先验配置，后启浏览器（R2 因果链簇）

- **aggregation_rule**: R2（因果链簇）——EK-01（固定生命周期）→ EK-02（preflight）→ EK-03（单 run 约束）→ EK-27（浏览器创建/停止计费）→ EK-28（key 解析）构成因果链：配置失败在浏览器创建前暴露，浏览器创建后只有错误状态，没有未捕获异常。
- **内容**: `_build()` 在 `browser.start()` 之前解析模型、key、provider pricing 和 prompt；`provider_preflight()` 只对官方 endpoint 做一次 GET；`BrowserConfigError`/`MissingAPIKeyError` 在 start 阶段 re-raise；其它 start 失败转为 `status='error'`、`steps=0`，并仍在 `finally` 停止浏览器。cloud browser 的 stop 负责结束计费。
- **解释范围扩大**: 长时/计费型 agent 的失败成本不对称：坏配置应尽早失败，运行中的 provider/browser 失败应保留可读结果。两者不能用同一种错误处理。
- **边界与反例**: preflight 不是完整健康检查；代理/未知 client/403 不阻断；模型调用仍可能在运行中失败。`run()` 之后不抛异常，调用方必须读 status。
- **回溯**: EK-01, EK-02, EK-03, EK-27, EK-28；证据 EV-19 至 EV-23, EV-83 至 EV-85。
- **层**: L3（Pattern）。
- **epistemic_status**: Validated Pattern。

## KO-05 终局交付是有限次修复的 best-effort，不是事务提交（R2 因果链簇）

- **aggregation_rule**: R2——EK-18（预算交付 turn）→ EK-19（一次有效性检查）→ EK-20（split delivery 按请求步拒绝）→ EK-26（checkpoint/partial 兜底）→ EK-30（status vs success）构成完整交付链。
- **内容**: 90% 预算进入 delivery-only turn；无效交付（占位/缺文件）触发一次 repair；同一响应多交付在该请求步被拒（代码注释称“once per run”，实现实际按 step 允许后续再次拒绝）；最后一次请求不再强拒；超时/最大步数时用 partial 或最后模型文本兜底。最终 `status` 表示循环如何结束，`success` 表示 agent 自评，二者不等价。
- **解释范围扩大**: 任何“必须给答案”的长任务，都要设计一个终止边界：在预算内尽量交付、有限修复、保留 partial、把“是否完成”与“是否结束”分开。
- **边界与反例**: 这不是“交付可靠”。占位/缺文件在最后一次请求会被接受；非占位但事实错误的答案也不会被拦；一次 repair 可能不足以修正结构。
- **回溯**: EK-18, EK-19, EK-20, EK-26, EK-30；证据 EV-26, EV-29 至 EV-36, EV-46。
- **层**: L3（Pattern，项目内强证据 + 测试契约）。
- **epistemic_status**: Validated Pattern（限定本项目 loop 的交付边界）。

## KO-06 状态、任务成功、验证结论是三套独立信号（R3 不变量簇）

- **aggregation_rule**: R3——EK-21（judge 不改变 status）→ EK-30（status/success 分离）→ EK-34（测试契约）共同约束：“一个 run 的结束状态、agent 自评、外部 judge 不能合并成一个布尔值”。
- **内容**: `status` 是 `completed|timeout|max_steps|error`；`success` 来自 `finish(success=...)`；`judgement` 是可选 judge 的结论。judge 失败只让 `judgement=None`，token/cost 仍计入；`completed` 仍可能 `success=False`；`max_steps`/`timeout` 可能什么都没交付。
- **解释范围扩大**: Agent 评测/监控若只读一个 `success` 布尔值，会同时丢失“是否跑完”“是否自认成功”“第三方是否认可”三种信息。三者应分别存储、分别报警。
- **边界与反例**: judge 与 agent 可能共享同一盲点；judge 是 advisory，不是 gate；success 可被模型错误自报。三信号独立不等于三信号可靠。
- **回溯**: EK-21, EK-30, EK-34；证据 EV-24, EV-69 至 EV-75, EV-34。
- **层**: L3（Pattern）。
- **epistemic_status**: Validated Pattern（限定本项目结果对象与 judge 路径）。

## KO-07 Compaction 把“证据”留在 prompt 之外（R2 因果链簇）

- **aggregation_rule**: R2——EK-22（触发）→ EK-23（投影 + archive）→ EK-24（失败保留全量）→ EK-25（图片批量裁剪）构成链：压缩是“投影”，不是“删除证据”。
- **内容**: 触发阈值为 75% char budget 或 65% context window；投影保留 summary checkpoint、最近两个 assistant/tool 组、用户消息原文；被覆盖消息写入 0600 的 redacted archive；投影不变短则拒绝压缩；summary 失败保留全量并在后续请求重试；图片按批次裁剪以保护 prompt cache。
- **解释范围扩大**: 长上下文系统的“压缩”应区分“模型看到什么”和“系统保留什么”。只做前者会把审计证据一起删掉；只做后者会继续超限。
- **边界与反例**: archive 的保留期由 workspace 生命周期决定；它不是不可变审计日志；summary 仍是“generated, fallible reference”，原文用户请求优先。provider 仍可能因估算误差拒绝。
- **回溯**: EK-22, EK-23, EK-24, EK-25；证据 EV-76 至 EV-80。
- **层**: L3（Pattern）。
- **epistemic_status**: Validated Pattern（限定本项目 Compaction 实现与测试）。

## KO-08 同一产品，两个权威平面：Agent 与 MCP 不可互换（R4 主题簇）

- **aggregation_rule**: R4（主题簇）——EK-16（MCP 同进程 exec）→ EK-17（cloud 拒绝 admin）→ EK-33（CLI/MCP 表面）→ EK-08/09（Agent worker 隔离）覆盖“同一能力的多入口”主题的互补维度。
- **内容**: Agent 路径：独立 worker 进程组、env allowlist、`sensitive_data`/`type_secret`、checkpoint redaction、per-cell timeout。MCP 路径：同进程 `exec`、无 secrets、无 worker env allowlist、无独立进程组；cloud 模式在有 domains 时拒绝 admin 调用，但并非全流量 Guard。两者共用 `browser_exec` 名称，却不共用权威模型。
- **解释范围扩大**: 多入口产品必须按“入口 → 执行平面 → 策略层”画权限图。同名工具不代表同等隔离；CLI/MCP/Agent 是三条不同的信任边界。
- **边界与反例**: MCP 也可以附着用户 Chrome 或启动自有 Browser；其行为取决于启动方式和是否配置 domains。不能把所有 MCP 调用一律说成“无 Guard”。
- **回溯**: EK-08, EK-09, EK-16, EK-17, EK-33；证据 EV-15, EV-38, EV-92 至 EV-94。
- **层**: L3（Pattern）。
- **epistemic_status**: Validated Pattern（限定本项目 Agent/CLI-MCP 两条路径）。

## KO-09 迁移用显式错误替代静默降级（R4 主题簇）

- **aggregation_rule**: R4——EK-32（legacy 名称错误）→ EK-33（CLI/MCP 表面迁移）→ EK-04（执行单元变化）覆盖“版本迁移”主题：旧语义不再可用时，系统选择报错而不是假装兼容。
- **内容**: `legacy.py` 安装 meta_path finder，访问 0.x 模块/名称时抛统一 removed error；`Agent.__getattr__` 指向 removed names；0.x planner/DOM/replay/pause/stop/callbacks/script export 被删除；`--mcp` 提示 deprecated 并服务新工具；旧 MCP 工具名返回 removed error 并指向 `browser_exec`。
- **解释范围扩大**: 破坏性升级的安全做法不是“保留一个会误用的兼容层”，而是“在第一次访问时明确失败，并给出替代路径”。静默 fallback 会让调用方以为旧约束仍生效。
- **边界与反例**: 这只覆盖名称/导入兼容，不覆盖旧配置语义；少数 tuning knob 只 warning 后忽略，不是全部显式失败；`--mcp` 仍可启动，但语义已换。
- **回溯**: EK-32, EK-33, EK-04；证据 EV-12, EV-15, EV-90, EV-92。
- **层**: L3（Pattern）。
- **epistemic_status**: Validated Pattern（限定本项目 0.x→1.0 迁移）。

## KO-10 认知模型：浏览器 agent 运行时像一个小型 OS（L4）

- **aggregation_rule**: R4——KO-01（进程/代码执行）+ KO-02（策略层）+ KO-07（journal/archive）+ KO-05（终止边界）抽象为同一稳定关系：运行时 = 执行进程 + 策略层 + 证据日志 + 终止协议。
- **内容**: **浏览器 agent 的可靠性不取决于模型多聪明，而取决于运行时是否把“执行、授权、证据、终止”分成四个可分别观察的平面。** 在 browser-use 1.0 中：执行平面是 REPL worker；授权平面是 domain Guard/secret scope；证据平面是 cell output/screenshot/checkpoint/compaction archive；终止平面是 delivery turn/status/partial。
- **解释范围扩大**: 可解释为什么“换一个更强的模型”不会自动修复越权、丢证据、超时无结果；这些是运行时平面的问题，不是推理能力的问题。
- **边界与反例**: 这是模型，不是已证明的通用定律。单项目证据不能证明所有 agent 都必须长成这样；短任务、无状态工具调用可能不需要四平面。MCP 路径就刻意省掉了执行/授权平面的一部分。
- **回溯**: KO-01, KO-02, KO-05, KO-07；EK-04 至 EK-26, EK-33。
- **层**: L4（Cognitive Model）。
- **epistemic_status**: Hypothesis（单项目抽象；跨项目未验证）。

## KO-11 认知模型：可靠性是“可恢复进度”，不是“事务正确”（L4）

- **aggregation_rule**: R2——KO-04（前置失败）+ KO-05（best-effort 交付）+ KO-07（证据保留）+ KO-09（显式迁移）收敛为稳定关系：系统优化的是“在不确定执行中保留可恢复的进度”，不是“每一步都原子正确”。
- **内容**: **长时 agent 的正确性单位不是单步事务，而是可恢复的检查点。** browser-use 用 checkpoint/partial 保留可读结果，用 compaction archive 保留被压缩证据，用 status/success/judge 三信号避免把“结束”误当“成功”，用显式迁移错误避免旧语义残留。
- **解释范围扩大**: 可解释为什么“失败后重跑整个任务”不如“保留 partial + 指出已完成/缺失部分”；也可解释为什么 judge 不应成为新的失败源。
- **边界与反例**: 这不是“部分结果总是优于重试”。对支付、提交表单、删除数据等有外部副作用的动作，可恢复进度不能替代幂等/补偿；browser-use 的 checkpoint 也不保证外部动作只发生一次。
- **回溯**: KO-04, KO-05, KO-07, KO-09；EK-18 至 EK-26, EK-32。
- **层**: L4（Cognitive Model）。
- **epistemic_status**: Hypothesis（本项目强证据；跨项目/副作用场景未验证）。

## KO-12 方法论：按“执行点 / 恢复层 / 证据面”设计浏览器 agent（L5，跨项目待验证）

- **aggregation_rule**: R4——综合 KO-01 至 KO-09 的可操作化，不新增事实。
- **内容**: 构建/审计浏览器 agent 时按三张表推进：
  1. **执行点表**：模型能做什么？是动作枚举还是代码？代码在哪里跑（独立进程/同进程）？超时杀谁？
  2. **策略层表**：每个入口（Agent/CLI/MCP/attached Chrome/cloud）分别启用哪些层？断开/例外时是 fail-open 还是 fail-closed？
  3. **证据面表**：模型看到什么、系统保留什么、遥测发送什么、redaction 覆盖哪些通道、哪些通道明文残留？
  最后才问“模型能力/提示词/benchmark 分数”。三张表没有填清楚，分数不能证明治理成立。
- **解释范围扩大**: 可直接用于审计其他浏览器 agent、桌面 agent、终端 agent；也可用于写产品安全声明，避免把“功能名”当“执行层”。
- **边界与反例**: 这是方法论提案，不是经跨项目验证的 Principle。它可能对短生命周期、无外部副作用的任务过重；也可能低估模型层面的注入风险。
- **回溯**: KO-01 至 KO-09；EK-04 至 EK-35。
- **层**: L5（Methodology）。
- **epistemic_status**: Cross-project validation pending（明确不写成已验证 Principle）。

## KO 聚合矩阵

| KO | 规则 | 簇内 EK | 主导边 | 层级 | 认知状态 |
|---|---|---|---|---|---|
| KO-01 | R1 | EK-04/05/06/07/08/09 | mechanism | L3 | Validated Pattern |
| KO-02 | R3 | EK-12/13/14/15/17 | constraint | L3 | Validated Pattern |
| KO-03 | R3 | EK-09/10/11/28/31 | constraint | L3 | Validated Pattern |
| KO-04 | R2 | EK-01/02/03/27/28 | causal | L3 | Validated Pattern |
| KO-05 | R2 | EK-18/19/20/26/30 | causal | L3 | Validated Pattern |
| KO-06 | R3 | EK-21/30/34 | contrast | L3 | Validated Pattern |
| KO-07 | R2 | EK-22/23/24/25 | causal | L3 | Validated Pattern |
| KO-08 | R4 | EK-08/09/16/17/33 | subsystem/contrast | L3 | Validated Pattern |
| KO-09 | R4 | EK-04/32/33 | contrast | L3 | Validated Pattern |
| KO-10 | R4 | KO-01/02/05/07 | abstraction | L4 | Hypothesis |
| KO-11 | R2 | KO-04/05/07/09 | abstraction | L4 | Hypothesis |
| KO-12 | R4 | KO-01 至 KO-09 | synthesis | L5 | Cross-project pending |

## 配比与反退化检查

- EK：35；KO：12（目标 7–12 上限）。
- L3：9；L4：2；L5：1。
- 每个 KO 至少 3 个 EK；无悬空升维。
- 没有“同一子系统”直接当聚合规则：KO-01 是机制链，KO-02 是不变量，KO-04/05/07 是因果链，KO-08/09 是主题互补。
- 两个 L4 都标 `Hypothesis`，没有把单项目模型写成 Principle。
- L5 明确 `Cross-project validation pending`，不冒充已验证方法论。
