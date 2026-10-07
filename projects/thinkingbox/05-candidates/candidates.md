# 05 · Candidates — 未验证假说 / 跨项目假说 / 不确定结论

> 本文件存放**不能确认为仓库内 Fact** 的内容：外部宣称（provenance 标注）、跨项目假说、暂定模式、不确定结论。Candidates 与 KO 严格区分：KO 是仓库内可回溯的 Pattern/Model，Candidates 是 Hypothesis（含验证路径）。**Hypothesis 不冒充 Fact；Cross-project Candidate 不写成已验证 Principle。**

---

## A. 外部宣称隔离区（provenance = arXiv:2608.19741 / 新闻 / README 转述，非本仓可验证）

> 以下数字来自论文与新闻对 thinkingbox-data 语料的分析——本仓库只是评测框架，这些统计的原始数据在**独立仓库 thinkingbox-data**，本 run 无法在仓库内验证，故全部标记为外部宣称。引用它们描述 ThinkingBox 时必须带 provenance。

| 编号 | 宣称 | provenance | 本仓可验证性 |
|---|---|---|---|
| C-01 | 121,680 trials × 12 模型 | arXiv:2608.19741 / 新闻 | 不可验证（数据在 thinkingbox-data） |
| C-02 | 79,853 次未过可执行检查 | 同上 | 不可验证 |
| C-03 | 67.24% 失败仍干净结束无报错 | 同上 | 不可验证 |
| C-04 | 79.9% 失败源于工具处理 | 同上 | 不可验证 |
| C-05 | 每任务 20 次连续执行 | 同上 | 部分可验证（本仓 `--repeat` 机制支持任意重复次数，默认值取决于调用方；"20 次"是语料实验配置，非框架默认） |
| C-06 | pass@1 幻觉终结 | 同上 | 机制可验证（EK-38/39），结论数字不可验证 |

## B. 跨项目假说（Hypothesis，cross-project validation pending）

### C-07 · "状态真相评分"在非 MCP 环境也可行
- 假说: effects 取证协议（__reserved__init/geteffects）不依赖 MCP 特有能力——任意有状态环境（数据库、文件系统、API 后端）都可实现"声明初始状态 + 取证当前状态"的评测面。
- 当前证据: 本仓 MCP server 形态（S3 已实现）；tests/servers.yaml 四形态 server。
- 缺失证据: 非 MCP 环境的实现样例。
- 验证路径: 在 langgraph（已有 corpus）或其他 agent 框架上实现同构 effects 协议并比较。

### C-08 · pass^k 指标对"偶发失败型 agent"的区分度优于 pass@1
- 假说: (c/n)^k 在 k 较大时能显著放大偶发失败（95%×95%×...），比 pass@1 更能暴露不稳定 agent。
- 当前证据: 数学上显然（本仓 EK-38 注释论证）；但无本仓内 benchmark 数据支撑。
- 缺失证据: 真实语料对比实验（在 thinkingbox-data）。
- 验证路径: 用本仓框架跑 baseline vs candidate，比较 pass^1 与 pass^20 的排序变化。

### C-09 · 注入防护三件套（sanitize/untrusted 声明/编码）可迁移到任意 LLM-as-judge 管道
- 假说: EK-23/43/44 的三种防护机制组合，是 LLM 评测管道注入防护的通用最小集。
- 当前证据: 本仓三处独立实现（S4 实现级）。
- 缺失证据: 在其他评测框架（agentevals/deepseek-harness，已有 corpus）中的对照。
- 验证路径: 对比 agentevals 的 judge 注入防护面。

### C-10 · 评测可复现性组合（配置快照 + 决策动机 + 可重放产物）是 agent 评测的通用最佳实践
- 假说: run_metadata.yaml + judge_motivation + JSONL 全量留痕的组合，能显著提升评测结论的可信度与可审计性。
- 当前证据: 本仓全链路实现。
- 缺失证据: 跨框架定量对比（如"有留痕 vs 无留痕的评测结论复查成本"）。
- 验证路径: 与 deepseek-harness（已有 corpus）的痕迹断言留痕度对比。

## C. 暂定模式 / 不确定结论（本仓内可复现但未达 Pattern 强度）

### C-11 · 子进程 \0 分隔协议是轻量 IPC 的稳健模式（暂定）
- 观察: TestScriptSubprocess 用 \0 分隔响应与噪音（EK-03）——\0 非合法 JSON 字符，可靠。
- 为何暂定: 单实例（本仓）；未见他处系统化使用。
- 潜在推广: 任意"子进程输出含噪音需提取结构化响应"的场景（CLI 工具封装）。

### C-12 · 解码/测试解耦（run-test 重跑）的成本收益边界未量化
- 观察: run-test 从 DecodeResult 重跑测试，避免重跑 agent（EK-08）。
- 不确定: TestContext 快照的"完整性"在多大程度上等价于真实会话（如动态状态缺失时的误判率）。
- 结论: 机制成立（S3/S4），收益量化待 benchmark。

### C-13 · tag taxonomy 部署注入模式（gitignored 真实值 + 公开示例回退）的正确性依赖发布纪律
- 观察: tag_types 加载 gitignored taxonomy，缺失回退 example（EK-50）。
- 不确定: 若真实 taxonomy 与 example schema 漂移（字段名变更），fail loudly 是否足够。
- 结论: 机制清晰，漂移风险待观察（D 级，项目局部）。

## D. 已排除/不成立

### C-14 · "框架默认每任务 20 次连续执行"——不成立
- 结论: 本仓 `--repeat` 参数由调用方控制，无默认 20 的框架证据（C-05 中"20 次"是语料实验配置，非框架默认）。外部宣称的"每任务 20 次"不能推断为框架行为。

### C-15 · "pass^k 与论文 pass^k 同名同义"——部分成立
- 结论: 本仓 agg_main.pass_power_k 即论文 pass^k 的实现源（命名/注释对应）；但论文统计结论（幻觉终结等）依赖 thinkingbox-data，本仓无法独立复现完整结论。
