# 06 · Validation & Evidence — dart_agent_core

> 六 Auditor 视角 + 先 Blind Reconstruction。结论如实，不弱化证据标准换全绿。

## 1. Blind Reconstruction（盲重建——不把考古产物当事实来源）

独立重读关键文件，重建以下结构，再与 EK/KO 对照：

**重建 A：主循环骨架**（独立读 `stateful_agent.dart` + `doc/architecture.md`）
→ 重建结果：`runStream` 含 4 阶段（prepare → LLM call → tool exec → persist）+ 双层计数（total/current）+ 3 重 hook 组（model/tool/persist）+ 取消复查 + finally 三连（afterRun→persist→MCP disconnect）。**与 EK-01/02/05/11/37 一致**。

**重建 B：评测管道**（独立读 `eval_runner.dart` + `recording/replay/rate_limit_gate`）
→ 重建结果：queue → concurrency 信号量 → per-trial prepare → run(timeout) → shouldGrade → grade → dispose → recorder.snapshot → export → reportStore。**与 EK-45/46/48/49/52 一致**。

**重建 C：工具执行参数装配**（独立读 `_executeTools`）
→ 重建结果：JSON schema properties 顺序迭代 → 类型铸造（array/int/number）→ positional pad null → Function.apply/直接调用 → 返回后取消复查 → catch 转 isError。**与 EK-13/14/15/16/17 一致**。

**重建 D：循环检测**（独立读 `loop_detector.dart`）
→ 重建结果：签名环形缓冲（5 同签名）→ LLM 诊断（30 后每 10、置信 0.8、历史清洗）。**与 EK-38/39/40 一致**。

## 2. Truth Auditor（这句话是真的吗？）

| 断言 | 判定 | 证据 |
|---|---|---|
| pass@k 用 Codex 无偏组合公式 | ✓ TRUE | `pass_at_k.dart` 注释 + metrics_test 数值用例（n=4,c=2,k=2→0.8333）|
| pass^k 是经验估计器且注释说明动机 | ✓ TRUE | `pass_caret_k.dart` 注释原文 |
| cacheSalt 独立于 runName | ✓ TRUE | `trial.dart:cacheSalt` 注释 + recording client 文档 |
| strictReplay 默认 true | ✓ TRUE | `replay_llm_client.dart` 构造默认值 |
| judge 校准默认 tolerance 0.15 / top20 | ✓ TRUE | `judge_calibrator.dart:CalibrationConfig` |
| 只对 completed trial 评分 | ✓ TRUE | `eval_runner.dart:_runOneTrial` shouldGrade 注释 |
| MCP 连接 per-run 生命周期 | ✓ TRUE | `stateful_agent.dart` finally + `doc/architecture.md` |
| "Suspend" 消息触发挂起可 resume | ✓ TRUE | `doc/architecture.md` + cancel 测试 |
| 无 CI 配置文件 | ✓ TRUE | 仓库根无 .github/workflows / gitlab-ci（快照核对）|
| 两公开入口解耦 | ✓ TRUE | `AGENTS.md` 原文 + lib 结构 |

## 3. Coverage Auditor（还有什么重要东西没发现？）

**强制子系统覆盖检查**：

| 子系统 | 覆盖 | 说明 |
|---|---|---|
| agent loop / hooks | ✓ EK-01~11, KO-01/05/06 | 全读 |
| tools | ✓ EK-12~18 | 全读 |
| state/memory | ✓ EK-19~24 | 全读 |
| skills | ✓ EK-25~29 | 全读 |
| sub-agent | ✓ EK-30~33 | 全读 |
| MCP | ✓ EK-34~37 | mcp_manager 全读；mcp_session 未读（mcp_dart 封装层）|
| loop detection | ✓ EK-38~40 | 全读 |
| eval core | ✓ EK-41~52 | 全读核心；graders/code+human、reporting 细节未全读 |
| platform/providers | ✓ EK-53~56 | 架构面；各 provider client 正文未逐行读 |
| governance | ✓ EK-57~60 | AGENTS.md + architecture.md 全读 |

**已知未覆盖（诚实披露）**：`message.dart`（593 行，多模态 content parts 细节）、`mcp_session.dart`（mcp_dart 封装）、eval 组装面（suite_loader/json_eval_task/grader_registry/code_grader/human_grader/score）、reporting（diff_reporter/report_generator）、observability（langfuse_client）、suite_health 阈值常量文件、其余 39 个测试文件。**这些不改变 EK/KO 结论**（核心机制已从主路径实读），但 Flow Atlas 的细粒度边可能需补证。

## 4. Flow Auditor（Flow Atlas 是否真的对应代码？）

抽查关键 Edge：
- `beforeToolCall deny → syntheticResults 回流`：✓ 实读 `_executeToolCallPhase`，deny/defer 结果写入 syntheticResults 且 skip 执行。
- `properties.keys 迭代 + pad null`：✓ 实读 `_executeTools` 参数装配。
- `shouldGrade=passed||failed`：✓ 实读 `_runOneTrial`。
- `finally: afterRun→persist→mcp disconnectAll`：✓ 实读 `runStream` finally。
- `clone worker 最近 10 条快照`：✓ 实读 `_copyParentHistory`（limit 10）。
- **未验证边**：recording hash 的精确字节级格式（llm_request_hash.dart 未逐行读）、Tpm 令牌桶的 refill 时序细节——列为 Level-2 待补。

## 5. Abstraction Auditor（是否过度升维？）

| KO | 层级判定 | 依据 |
|---|---|---|
| KO-01 Pattern | 保持 | hook 类型化控制面在本项目跨 3 子系统出现；跨项目未验证 → 不升 Principle |
| KO-02 Pattern | 保持 | 评测因果链是本项目内部完整；跨项目推广需 agentevals 对照 |
| KO-03 Principle (pending) | **降级处理** | 取消优先已在两子系统（tool/worker）+ thinkingbox cancel 语义观察；但 cross-project 未系统验证 → 保留 pending 标注，不写 Law |
| KO-04 Pattern | 保持 | 上下文四策略完整但单项目证据 |
| KO-05 Principle (pending) | **降级处理** | "失败不伪装"跨 agent/eval/sub-agent/judge 四子系统是项目内最一致不变量；跨项目证据（thinkingbox/opencode 同族）为观察级 → 保留 pending |
| KO-06 Pattern | 保持 | 两级检测是项目内设计；跨框架对照未做 |
| KO-07 Pattern | 保持 | 复现性机制两子系统；跨项目未验证 |
| KO-08 Pattern | 保持 | local-first 工程形态单项目证据 |

**升维铁律核对**：无 KO 宣称跨项目已验证 Law；所有 Principle 标 pending。

## 6. Counterexample Hunter（反例预算制：每个 L3+ KO ≥3 定向攻击）

| KO | 反例 1 | 反例 2 | 反例 3 | 结果 |
|---|---|---|---|---|
| KO-01 | `AgentController.request` defaultValue 是控制旁路 | RunJavaScript 检查可被 `..` 路径绕过吗？（rootWithSep 前缀匹配——需测 `../` 越界）| hook 顺序固定，无优先级注入 | 反例 1 削弱"唯一通道"表述 → 已改为"唯一行为控制通道"；反例 2 未验证（记 C-09）；反例 3 无证据 |
| KO-02 | runner_e2e 用 Stub 无真实 LLM——闭环可不经真实模型 | 录音仅成功对——失败路径不可重放 | suite health 依赖 reportStore 持久化（未实读存储实现）| 反例 1 是设计特性非缺陷；反例 2 记录为边界；反例 3 记为 Level-2 待补 |
| KO-03 | Suspend 挂起 ≠ 取消——resume 后取消纪律是否重放 | worker 抛非 AgentException 的 Error 也吞吗？| MCP disconnectAll 在取消路径是否执行 | 反例 1 是语义分化（挂起非终止）；反例 2 catch 面未逐行核对（记待补）；反例 3 finally 保证 ✓ |
| KO-04 | 无 compressor 时 loop 合法（治理可选层）| retrieve_memory 找不到 snapshot 返回错误文本（无自动回退）| MCP Layer 1 与技能四节是两套披露实现（机制同源但代码独立）| 反例 1 记录边界；反例 2 确认行为（明确错误文本）✓；反例 3 强化机制簇判定 |
| KO-05 | defer 默认 isError=false——被延迟动作不标错 | grader 异常 → Score(value:null) 也走 null 通道 | 空响应重试 3 次后 loopDetection——上限是否过严 | 反例 1 是"不伪装失败"的镜像（不伪装成功也不伪装失败）✓ 已写入 KO-05 边界；反例 2 确认 ✓；反例 3 无证据 |
| KO-06 | 流式部分轮不触发 LLM 诊断（"tool calls are actionable progress"）| 签名环形缓冲仅 5——长周期循环漏检 | confidence>0.8 是硬阈值——诊断失败降级 false | 反例 1 确认 ✓；反例 2 记录边界；反例 3 确认 ✓ |
| KO-07 | systemPromptHistory 只在 hash 变化时记录——同内容不同序算变化吗？| recording 无压缩——大响应录音膨胀 | replay strict 模式无 fallback 时 cache miss 即失败（CI 可用性风险）| 反例 1 未验证（hash 算法细节待补）；反例 2 无证据；反例 3 是显式设计（forcing re-recording）✓ |
| KO-08 | 无 CI 配置依赖开发者自觉 | Web 端 localStorage API key 是明文安全边界 | 6 provider 未覆盖本地 LLM（Ollama 走兼容面）| 反例 1 已进 C-07；反例 2 记录安全边界；反例 3 确认 ✓ |

**新增候选（反例审计产生）**：C-09：RunJavaScript 路径前缀匹配的 `../` 越界行为未验证（rootWithSep 前缀——需确认绝对路径规范化后仍被前缀约束）。

## 7. Epistemic Auditor（认知状态标注是否诚实）

- **Fact/Observation/Hypothesis/Pattern/Principle 逐条核对**：EK 层全部 S3/S4（已实现/测试验证），无 S0-S2 混入；KO 层 6 Pattern + 2 Principle(pending)；Candidates 全部 Hypothesis。
- **无"未验证假设冒充知识"**：C-01~C-08 均标 Hypothesis/未决，未进 KO。
- **无跨项目 Candidate 写成已验证 Principle**：C-03/C-08 明示"缺失证据/验证路径"。
- **systemPromptHistory 的 hash 算法**（content.hashCode + tool names join sort）已实读——非 LLM 语义 hash，是确定性字符 hash（Epistemic 标注：确定性实现，S3）。

## 8. Contradictions & Counterexamples 保留（不删除）

- **Contradiction-1（记录）**：注释 "maxTurns uses currentLoopCount; empty/hook retries already returned" 与测试 "empty stopReason retry does not consume maxTurns budget" 一致——无矛盾。
- **Contradiction-2（记录）**：`ER` 注释 "同 isolate 单线程需原子临界区" vs RateLimitGate 的 Completer 队列——两者一致（并发模拟 + 限速独立），非矛盾。
- **Counterexample 记录**：见 §6 各 KO 反例（反例 1/2/3 全部保留在案，不因"不漂亮"删除）。

## 9. Reconciliation 摘要（供阶段⑥合并）

- 独立盲重建与考古产物无实质性冲突（重建 A/B/C/D 全部一致）。
- 已修正表述：KO-01 "hook 是唯一治理通道" → "hook 是唯一**行为**控制通道"（controller.request 旁路属查询型）。
- 已记录待补（Level-2）：mcp_session/message.dart/eval 组装面/阈值常量/39 测试文件/hash 细节/`../` 越界——写入 C-09 与 §3。

## 10. 质量指标（供 run_metadata / PR）

| 指标 | 值 |
|---|---|
| Facts/Evidence 数 | 60 EK（全部带证据引用，S3-S4）|
| EK 平均出边 | ≥1（links 覆盖率 100%）|
| 游离 EK | 0 |
| KO 数（aggregation_rule 覆盖）| 8（R1×3 / R2×2 / R3×2 / R4×2，覆盖率 100%）|
| KO 平均簇规模 | 5.4（3~10）|
| Candidates | 8（全部 Hypothesis/未决）|
| Counterexample 攻击 | 8 KO × 3 = 24 次（含 2 个未验证反例 → C-09 与 §3）|
| Contradiction | 0 实质矛盾（2 条记录在案）|
| 盲重建 | 4 结构全一致 |
| Validation 结论 | PASS（含 2 条 Level-2 待补披露）|
