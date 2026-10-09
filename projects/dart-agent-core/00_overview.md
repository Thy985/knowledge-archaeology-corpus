# 00 · Overview — dart_agent_core

- **job_id**: ARCH-2026-10-10-001
- **project**: dart_agent_core（memex-lab/dart_agent_core）
- **repository**: https://github.com/memex-lab/dart_agent_core.git
- **commit**: `43c11f1b9a00a554c27c8ad1839175c2dd0e4a91`（2.1.7, 2026-10-09）
- **skill_version**: knowledge-archaeology v3.2
- **mode**: initial

## 一句话定位

Dart/Flutter 生态中第一个可深度考古的 **mobile-first / local-first agent 运行时 + agent 评测子系统一体化库**：`StatefulAgent` 完整 think-act-observe 循环（工具调用/流式/状态持久化/技能/子代理/规划/上下文压缩/循环检测/MCP），加 `eval.dart` 评测子系统（suite/trial/grader/transcript/record-replay/pass@k/pass^k/judge 校准/suite 健康），无需 Python/Node 后端，6 平台 + WASM。

## 核心认识（本考古最重要的三件事）

1. **评测与运行时共享同一"非确定性保护"心智**：`pass@k/pass^k` 测的就是非确定性下的成功率，而 record/replay 的 `trialSalt`（Trial.cacheSalt = `taskId#trialIndex`，独立于 run name）刻意保留每 trial 独立缓存槽，注释明言"否则框架静默摧毁 pass^k/pass@k 本该测量的非确定性"——评测统计设计与可复现基础设施是**同一设计的正反两面**（EK-41/42/45/46）。
2. **失败不伪装成功，是全库最一致的不变量**：空模型响应连续 3 次即 loopDetection（不消耗 maxTurns 预算，EK-03）；评测只对 completed trial 喂 grader（timeout/error 的占位 outcome 不得被弱 grader 误判通过，EK-48）；worker 子代理的局部失败一律是工具结果而非 run 失败（仅共享取消逃逸，EK-33）；judge 无法判断必须返回 null 而非编造分数（EK-44）。这一不变量跨 agent 运行时、评测、子代理、judge 四个子系统重复出现。
3. **工具面=控制面解耦，hook 是唯一的治理通道**：AgentHook 10 阶段类型化控制面（proceed/respond/retry/deny/defer/stop/continue/abort），工具调用前可 deny/defer/改写，状态持久化前可 skip/abort，模型调用前可合成响应——治理不进工具实现本身（EK-07~11）。

## 交付物清单

| 产物 | 文件 |
|------|------|
| Project Layer | `01_project_layer.md` |
| Engineering Knowledge（EK Graph） | `02_engineering_knowledge.md` |
| Knowledge Layer（KO，aggregation_rule） | `03_knowledge_layer.md` |
| Flow Atlas（七类流） | `04_flow_atlas.md` |
| Candidates | `05_candidates.md` |
| Validation & Evidence | `06_validation.md` |
| run metadata | `run_metadata.yaml` |

## 证据底座规模

- 精读源码：stateful_agent.dart（2015 行全读）、agent_hook.dart（684 全读）、eval 核心（pass_at_k/pass_caret_k/judge_calibrator/eval_runner/recording/replay/rate_limit_gate/suite_health/model_grader/trial 全读）、skill.dart/sub_agent.dart/context_compressor.dart/planner.dart/memory.dart/loop_detector.dart/mcp_manager.dart/llm_client.dart/tool.dart/controller.dart 全读
- 测试抽查：metrics_test/calibration_test/runner_e2e_test/stateful_agent_loop_test
- 文档：README、AGENTS.md、doc/architecture.md
- 规模：lib 17,176 行 / test 12,072 行 / 41 测试文件
