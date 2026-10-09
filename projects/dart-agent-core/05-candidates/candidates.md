# 05 · Candidates — dart_agent_core

> 不能确认的内容留在候选。与 KO 严格区分：这里全是 Hypothesis / 暂定模式 / 跨项目假说，不冒充已验证知识。

## C-01 · 跨项目假说：pass^k empirical 选择动机的分化
- **观察**: dart_agent_core 注释明确选择 empirical `(c/n)^k`（"intuitive and consistent with how teams report X out of Y trials passed in CI"）；thinkingbox（10-08 考古）同样实现 empirical 版但动机为对抗 pass@1 幻觉/区分难例。
- **假说**: "同一估计器的两个独立项目各自给出不同但兼容的正当理由"→ 评测库生态对 pass^k 的选择反映**团队报告文化**而非统计正确性分歧。
- **缺失证据**: 无第三个独立项目的实现动机作对照；两项目均无线上 A/B 验证两估计器的行为差异。
- **验证路径**: 在 agentevals/deepseek-harness 评测层核对 pass^k 实现与注释动机。

## C-02 · 暂定模式：judge null escape hatch 是评测生态的隐性规范
- **观察**: dart_agent_core ModelGrader 要求 rubric 显式含 Unknown escape hatch（"Anthropic Step 5"）；judge_calibrator 将 null 排除在相关性外单独统计。
- **假说**: "LLM-as-judge 必须能被显式拒绝（null），且拒绝样本单列统计"正在成为 agent 评测框架的跨项目规范（dart 侧显式引用 Anthropic 步骤编号）。
- **缺失证据**: 仅本项目显式引用 Anthropic Step 5；尚未核验 agentevals/other 评测库是否同规。
- **验证路径**: 对照 thinkingbox/agentevals 的 grader 抽象与校准实现。

## C-03 · 跨项目假说：hook 类型化控制面与 controller 观察分离是 Dart agent 框架的特征签名
- **观察**: dart_agent_core 把"观察"（AgentController 事件，fire-and-forget）与"控制"（AgentHook 类型化结果）严格分离；`doc/architecture.md` 明言 controller "do not control the agent loop"。
- **假说**: "事件总线观察 + hook 控制"双通道是 agent 运行时框架的共同骨架（openai-agents/smolagents/langgraph 亦有 controller/hook 类比）。
- **缺失证据**: 未对上述框架逐一核对（各自命名与粒度不同）；dart 侧将此分离写入文档并加测试是本项目的强化证据。
- **验证路径**: 下一考古轮（opencode/hermes-agent）核对 hook/controller 分离设计。

## C-04 · 暂定模式：MCP 渐进披露（Layer 1）与技能渐进披露同构
- **观察**: MCP system prompt 只列服务器名+能力计数（EK-34）；技能 system prompt 四节且重 prompt 仅注入激活技能（EK-26）——同一"先列目录、按需展开"模式在两个子系统独立出现。
- **假说**: "能力目录渐进披露"是 agent 上下文治理的统一原语（MCP 服务器目录 / 技能目录 / 工具列表都适用）。
- **缺失证据**: 单项目内两实例（机制内聚），跨项目实例未见。
- **验证路径**: mcp corpus（已考古）与 opencode 的工具披露策略对照。

## C-05 · 未决观察：sub-agent 上下文注入的"最近 10 条"阈值
- **观察**: clone worker 复制父 history 最近 10 条快照（`_copyParentHistory` 保留 last 10）。
- **未决**: 10 条是否为经过调参的常数，或工程直觉；无测试显式验证该阈值的行为边界（sub_agent_test 是否覆盖长历史父代理未抽查）。
- **验证路径**: 读 sub_agent_test.dart 全文件 + git log 查该阈值引入提交。

## C-06 · 未决观察：评测 runner 单 isolate 并发
- **观察**: `ER:runSuite` 注释 "同 isolate 单线程需原子临界区"——并发是信号量模拟而非 isolate/线程。
- **未决**: 与 deepseek-harness/thinkingbox 的评测并发模型差异；Dart 单 isolate 限流是否成为大 suite 的扩展性边界。
- **验证路径**: 对照已考古评测类项目（thinkingbox/deepseek-harness）的 runner 并发实现。

## C-07 · 暂定模式：无 CI 文件 + 本地质量门的依赖
- **观察**: dart_agent_core 无 CI 配置文件；质量门 = pana/analyze/format/test（AGENTS.md）。
- **假说**: "文档纪律替代 CI"在小规模单人/双人维护的库中可行，但跨贡献者规模后失效。
- **缺失证据**: 项目无外部贡献者活跃度证据（14 forks/0 open issues）；无贡献者协作案例。
- **验证路径**: 后续考古观察其他无 CI 配置文件的活跃库。

## C-09 · 未决观察：RunJavaScript 路径前缀匹配的 `../` 越界行为（独立审计 F-2 产生）
- **观察**: `_runJavaScriptScript` 用 `fsAbsolutePath(scriptPath)` + `resolvedAbsolute.startsWith(rootWithSep)` 前缀匹配（rootWithSep = root + 分隔符）；rootPaths 为 `_normalizedSkillDirectoryPaths`（去重绝对路径集）。
- **未决**: `fsAbsolutePath` 是否对 `..` 段做规范化未验证——若返回未规范化路径，`<root>/../sibling.js` 仍以 `<root>/` 为前缀而逃逸白名单。Authority Flow 中 JS 执行白名单的完备性未闭合。
- **缺失证据**: `core/fs_io.dart:fsAbsolutePath` 实现未实读。
- **验证路径**: 后续轮读 fs_io.dart 实现 + 构造 `../` 越界测试用例。

## C-08 · 跨项目候选：Dart 生态 agent 框架位置
- **观察**: dart_agent_core 是目前 corpus 中唯一 Dart 原生 agent 框架（36+ slugs 无 Dart）；Tafcm（用户项目，Dart/Flutter）直接复用对象。
- **候选**: dart_agent_core 可作为 Dart 生态 agent 框架的**基准参考**（与 Python 生态 langgraph/opencode 对照的跨语言锚点）。
- **缺失证据**: 未探测其他 Dart agent 框架（langchain_dart/agent_dart 等）做生态定位。
- **验证路径**: KnowlegeMap 雷达新增 Dart agent 生态扫描。
