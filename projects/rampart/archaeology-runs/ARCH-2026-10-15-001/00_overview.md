# 00 Overview · RAMPART

- **run_id**: ARCH-2026-10-15-001
- **project**: RAMPART（Microsoft AI Red Team 官方）
- **repository**: https://github.com/microsoft/RAMPART.git @ `2c9067dff437deaa6562ec3979416ddda04f5a28`（2026-10-13）
- **skill_version**: knowledge-archaeology v3.2（EK Graph + R1-R4 聚合规则 + Blind Reconstruction）
- **一句话定位**: pytest-native 的 Agentic AI 安全测试框架——把"红队技术"变成可嵌入 CI 的安全测试。核心命题：**"安全测试框架如何让'不可观测'与'不安全'不被混淆，并让基础设施故障不被误报为安全发现"**。

## 考古摘要（本项目让我们认识到什么）
RAMPART 回答了一个尖锐问题：当你要给 AI agent 写自动化安全测试时，"看到了什么"和"没看到什么"必须被严格区分。三条主线：

1. **观测即契约（Observability as Contract）**：Adapter 声明 `observability_profile`（三档），评估器按通道保证取信——看不到工具调用时返回 UNDETERMINED 而非假的 NOT_DETECTED；SAFE 结论在 RESPONSE_ONLY + 零工具观测时被强制降级为 UNDETERMINED。"没看到 ≠ 没发生"。
2. **判定与故障分离（Verdict vs Infrastructure）**：基础设施异常（503、连接失败）统一转 `SafetyStatus.ERROR`，从攻击成功率分母剔除；xdist worker 崩溃导致的不完整运行强制非零退出——"green 必须意味着 ran and passed"。
3. **信任边界贯穿（Trust Boundary Everywhere）**：被测 agent 的输出（可能被注入的 response/tool_call/payload）在跨进程传输（xdist worker→controller）、渲染（terminal/report）、评估（LLM judge 收到的 transcript）三个环节都被当作不受信输入处理：JSON-safe 原语序列化 + schema/enum/depth 校验 + ANSI 剥离 + `safe_str` 防御 + judge system prompt 加固后缀。

## 关键数字（可追溯）
- 规模：132 .py / 30,891 行；rampart/ 包 10,501 行
- 内置攻击：1（XPIA，Cross-Prompt Injection Attack）；内置探测：1（Behavioral/SingleTurn）
- 内置评估器：4（ToolCalled / ResponseContains / SideEffectOccurred / LLMJudge）+ 组合算子 | & ~
- 依赖：pyrit==0.13.0 精确 pin；pytest>=9.0.3；pydantic>=2.13.4
- 测试：unit 全覆盖 + integration + scripts；coverage fail_under=80
- 判定优先级：attack DETECTED > UNDETERMINED > NOT_DETECTED；probe NOT_DETECTED > UNDETERMINED > DETECTED

## 产物清单
| 文件 | 内容 |
|---|---|
| 01_project-layer.md | 项目地图（L0 事实） |
| 02_engineering-knowledge.md | EK Graph（35 EK，六类边 + R1-R4 聚合） |
| 03_knowledge-layer.md | Generalized KO（8 个，含 aggregation_rule） |
| 04_flow-atlas.md | 七类流（Control/State/Data/Evidence/Authority/Memory/Policy） |
| 05_candidates.md | 未验证假说与跨项目候选 |
| 06_validation.md | Validation（Truth/Coverage/Flow/Abstraction/Counterexample/Epistemic + 反例） |
| run_metadata.yaml | run_id/commit/skill_version/quality metrics/validation result |

## 质量指标快照（详见 06）
- EK 35 条 / KO 8 个 / Candidates 6 条 / Flow Edge 28 条
- EK 平均出边 ≥1（满足）；游离 EK <20%（满足）；聚合规则覆盖率 100%
- Validation：CONFIRMED 22 / PARTIALLY_CONFIRMED 6 / OVER_GENERALIZED 1 / MISSING 1 / CONTRADICTED 0 / NEEDS_HUMAN_REVIEW 2
