# 01 Project Layer — OWASP Agent Memory Guard

## 1.1 项目地图（L0 事实，全部可追溯）
| 维度 | 事实 | 证据 |
|---|---|---|
| 语言/形态 | Python 库（单语言） | pyproject.toml |
| 版本 | 0.3.3（Beta，Apache-2.0） | pyproject.toml；README |
| 仓库 | OWASP/www-project-agent-memory-guard | GitHub |
| 定位 | "Runtime defense layer that protects AI agent memory from poisoning, tool abuse, privilege escalation, and excessive autonomy (OWASP)" | pyproject.toml description |
| 架构角色 | Guard 插在 agent 与其 memory store 之间，筛查每次操作 | README:90 |
| 核心依赖 | 仅 PyYAML>=6.0 | pyproject.toml |

## 1.2 入口
- **CLI**：`amg`（agent_memory_guard.cli:main）+ `amg-bench`（bench.cli:main）（pyproject [project.scripts]）
  - 子命令：`scan` / `serve`（HTTP server）/ `check`（单文本检查）（cli.py _build_parser）
- **API**：`MemoryGuard(policy=Policy.strict())`；`guard.write(key, value, source_class=...)` / `guard.read(key)` / `guard.delete(key)` / `guard.promote(...)` / `guard.baseline/verify/verify_all` / `guard.rollback(snapshot_id)`（guard.py）
- **MCP**：`amg-mcp-server` / `python -m amg_mcp_server`（FastMCP）
- **中间件**：`FastAPIGuard` / `FlaskGuard`（middleware.py）
- **GitHub Action**：action.yml（输入经 env 传递，防 run 脚本表达式注入 #122）

## 1.3 核心模块（src/agent_memory_guard/，51 py）
| 模块 | 行数 | 职责 | 关键符号 |
|---|---|---|---|
| guard.py | 768 | MemoryGuard 编排（write/read/delete/promote/baseline/verify/rollback） | `MemoryGuard.write` / `_run_detectors` / `_decide` / `_escalate` |
| classification.py | ~110 | MemoryClass 六类 + 晋升图 | `DEFAULT_PROMOTION_GRAPH` / `PromotionRules.requires_verification` |
| integrity.py | 60 | SHA-256 基线 | `canonical_serialize` / `hash_value` / `IntegrityRegistry.verify` |
| events.py | 107 | SecurityEvent + 枚举 | `SecurityEvent.to_dict` / `SourceClass` / `Action` / `Severity` |
| policies/policy.py | 371 | 声明式策略引擎 | `Policy.from_dict` / `PolicyRule.applies_to` / `Policy.decide` / `Policy.strict/tiered` |
| middleware.py | 140 | FastAPI/Flask 请求体扫描 | `FastAPIGuard` / `FlaskGuard` |
| detectors/ | 12 检测器 | 内容/语义/行为检测 | 见 1.4 |
| storage/memory_store.py | 65 | MemoryStore 协议 + InMemoryStore | `MemoryStore` / `InMemoryStore.snapshot/restore` |
| storage/snapshots.py | 91 | 快照 | `Snapshot`（digest 声明不验证）/ `SnapshotStore` |
| bench/ | 9 文件 | AMSB 基准 | `harness.ScenarioRun` / `scoring` / `scenarios.default_corpus` |
| integrations/ | 6 适配 | 框架接入 | `agno.py` / `autogen.py` / `crewai.py` / `langchain.py` / `llamaindex.py` / `openai_agents.py` |

## 1.4 检测器族（12）
| 检测器 | name | 机制 | 证据 |
|---|---|---|---|
| PromptInjectionDetector | prompt_injection | 11 条 regex（ignore/disregard previous instructions、jailbreak、system override 等），IGNORECASE\|DOTALL | detectors/injection.py |
| SensitiveDataDetector | sensitive_data | 13 类 secrets/PII regex（AWS/GitHub/OpenAI/Anthropic/Google/Slack/JWT/信用卡/SSN/email）+ redact | detectors/leakage.py |
| SizeAnomalyDetector | size_anomaly | max 64KB + 10× 增长因子 | detectors/anomaly.py |
| RapidChangeDetector | rapid_change | 写入速率异常 | detectors/anomaly.py |
| ProtectedKeyDetector | protected_key | glob fnmatchcase 匹配 protected 模式（仅 write） | detectors/protected_keys.py |
| CrossTaskContaminationDetector | cross_task_contamination | durable 类（tool_observation/retrieved_fact）跨任务读取标记（autogen#7673 scenario 1） | detectors/cross_task.py |
| SelfReinforcementDetector | self_reinforcement | 冷却（60s/3 次）+ 自相似（difflib ≥0.85）；trust-aware decay（默认仅 SYSTEM） | detectors/self_reinforcement.py；#124 |
| ExcessiveAutonomyDetector | excessive_autonomy | LLM08 过度自主模式（human_in_the_loop=false、max_iterations=inf 等） | detectors/excessive_autonomy.py |
| ToolAbuseDetector | tool_abuse | 伪造 tool_call 结构/外壳命令注入 | detectors/tool_abuse.py |
| MemoryPersistenceInjectionDetector | memory_persistence_injection | **延迟生效载荷**（"once stored/from now on" 持久化指令、canary token、伪造 prior-turn 标记） | detectors/memory_persistence_injection.py |
| PrivilegeEscalationDetector | privilege_escalation | role/permission/root 赋值注入 | detectors/privilege_escalation.py |
| MLInjectionDetector | ml_injection | DistilBERT/deberta-v3 分类（阈值 0.85，默认 protectai/deberta-v3-base-prompt-injection-v2，fallback deepset） | detectors/ml_injection.py |

## 1.5 生命周期
- 初始化：MemoryGuard 构造 → 合并 protected keys → 构造 3 个必需检测器（protected/cross_task/self_reinforcement）→ 默认检测器套件（7 个）→ 对已有 store 中 immutable 键建立完整性基线（guard.py __init__）
- 运行：write/read/delete 均过检测器+策略；事件累积 `guard.events`，handlers 实时回调
- 快照/回滚：`snapshot_on_block=True` 时 BLOCK 前 capture("pre-block")；`rollback(snapshot_id)` 恢复
- 发布：Release 0.3.3（#152）；下载统计自动 commit（analytics/）

## 1.6 配置与治理
- 策略：examples/policy.yaml 声明式（version 1 / default_action / protected_keys / immutable_keys / rules）
- 三预设：permissive（检测不拦截）/ strict（quickstart）/ tiered（按 key 命名空间映射不同 action——credentials.* block、facts.* quarantine）
- 治理：SECURITY.md / CONTRIBUTING.md / CODE_OF_CONDUCT.md / ROADMAP.md / CHANGELOG.md / CITATION.cff；OWASP Foundation 治理
- 合规：docs/compliance-mapping.md——NIST AI RMF 1.0（Govern/Map/Measure/Manage）+ EU AI Act 逐项映射

## 1.7 外部依赖
- 硬依赖：PyYAML（策略加载）
- 可选：fastapi/uvicorn/pydantic（serve）、transformers/torch（ML 检测器）、langchain-core/crewai/llama-index-core/pyautogen/openai-agents（集成适配）
- 测试/基准：无网络后端；AMSB adapter 抽象第三方记忆系统

## 1.8 权威机制
- 策略规则有序匹配（Policy.decide：首个 applies_to 生效，否则 default_action）——**规则顺序 = 权威优先级**
- 分类晋升图：唯一晋升路径，verified=True 显式用户 opt-in——**信任提升需显式验证**
- trusted_source_classes：仅配置类可衰减自增强计数（默认仅 SYSTEM）——**佐证权威默认不可信**
