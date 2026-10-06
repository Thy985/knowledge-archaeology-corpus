# Snapshot Artifact — ARCH-2026-10-07-001 · OWASP Agent Memory Guard

## 1. 快照信息
| 项 | 值 |
|---|---|
| run_id | ARCH-2026-10-07-001 |
| project | OWASP Agent Memory Guard（slug: agent-memory-guard） |
| repository | https://github.com/OWASP/www-project-agent-memory-guard.git |
| mode | initial |
| commit SHA | `84030912a8c1b96e42ada98d6ba2f4d4a7d345b1` |
| commit message | `chore: update download stats [skip ci]` |
| commit timestamp | 2026-10-06 12:45:44 +0000 |
| branch | main |
| version | 0.3.3（pyproject.toml；Release 0.3.3 #152） |
| license | Apache-2.0 |
| requires-python | >=3.9（classifiers 至 3.13） |
| clone | 浅克隆 + `--shallow-since=2026-08-01`，共 210 commits |
| 状态 | Development Status :: 4 - Beta |

## 2. 项目基础地图（事实全部来自仓库实际内容）

### 语言与形态
- Python 安全中间件库（单语言），发布名 `agent-memory-guard`；另有独立集成包 `langchain-agent-memory-guard` / `autogen-agent-memory-guard`、`amg-mcp-server`（MCP server）
- 定位（README:84-90）：**"Runtime defense layer that protects AI agent memory from poisoning, tool abuse, privilege escalation, and excessive autonomy"**——Guard 插在 agent 与其 memory store 之间，用检测器管线 + 声明式策略筛查每次操作

### 入口
- CLI：`amg`（agent_memory_guard.cli:main）+ `amg-bench`（agent_memory_guard.bench.cli:main）
- API：`MemoryGuard(policy=Policy.strict())`（README:61-63）
- MCP：`amg serve`（mcp-server/src/amg_mcp_server/server.py）

### 核心模块（src/agent_memory_guard/，51 py）
| 模块 | 职责 |
|---|---|
| guard.py | MemoryGuard 主类（拦截管线编排） |
| integrity.py | SHA-256 完整性基线 / Snapshot 快照 / 回滚；immutable_keys globs 匹配 |
| middleware.py | 中间件抽象 |
| detectors/（12 检测器） | anomaly / cross_task / excessive_autonomy / injection / leakage / memory_persistence_injection / ml_injection / privilege_escalation / protected_keys / self_reinforcement / tool_abuse（+ base） |
| policies/policy.py | Policy 声明式策略（YAML） |
| events.py | SecurityEvent 事件模型（含 detector 异常吞噬时也发事件 #138） |
| metrics.py / classification.py / exceptions.py | 度量 / 分类 / PolicyViolation 异常族 |
| integrations/ | agno / autogen / crewai / langchain / llamaindex / openai_agents 适配 |
| bench/ | AMSB（Agent Memory Security Benchmark）：adapter / harness / scenarios / scoring / report / cli / adapters.{baseline,third_party} |

### 根级资产
- mcp-server/（amg_mcp_server：__main__/server）
- scanner/（scan.py / rules.py / sarif_output.py——semgrep 静态扫描 + SARIF）
- semgrep/agent-memory-unguarded.py（semgrep 规则）
- benchmarks/security_benchmark.py
- examples/：secure_langgraph_memory.py / opentelemetry_hook.py / openai_agents_memory_guard.py / policy.yaml / quickstart / interactive_demo / record_demo
- integrations/：autogen-agent-memory-guard / langchain-agent-memory-guard（独立包）
- action.yml（GitHub Action）、docs/（architecture / detectors / benchmark / compliance-mapping / github-action / cli / api / guides）

### 核心数据结构
- Snapshot（integrity）：完整性快照（digest、键集）；**README/docs 明示 `Snapshot.digest is never verified`（#139 commit）——诚实声明安全边界**
- Policy：声明式策略（examples/policy.yaml；Policy.strict() 预设）
- SecurityEvent：统一事件（检测/拦截/异常）
- MemoryStore 协议：framework-agnostic 存储接口（GuardedChatMessageHistory 即其 LangChain 实现）

### 状态流
agent 操作（读/写 memory）→ Guard 检测器管线 → 策略判定（block/sanitize/report）→ SecurityEvent → 可选回滚到已知良好 Snapshot

### 测试体系
- tests/：41 py（含 detector 级、integrity 级、CLI/API、集成包测试 langchain/test_middleware.py）
- CI：GitHub Actions（ci.yml、codeql 上传 SARIF #146、workflow 供应链依赖 pinning #54；action 输入经 env 防脚本注入 #122）
- 基准：AMSB（benchmarks/security_benchmark.py + bench/ 全套）

### 配置与治理
- pyproject.toml：核心依赖仅 `PyYAML>=6.0`；extras：server(fastapi/uvicorn/pydantic)、ml(transformers/torch)、langchain/crewai/llamaindex/autogen/openai-agents
- SECURITY.md / CONTRIBUTING.md / CODE_OF_CONDUCT.md / CITATION.cff / ROADMAP.md / CHANGELOG.md
- OWASP www-project 治理（README 声明 OWASP Foundation）

### 权限与治理机制
- 策略层：Policy 声明（YAML/API）——拦截/放行/净化判定
- 集成层：drop-in middleware（LangChain middleware 默认 block on violation；OpenAI Agents 默认 HITL queue on block）
- 治理：GitHub Action（amg 扫描）、semgrep 规则（CI 静态防线）、SARIF 输出

### 外部依赖
- 运行时：PyYAML（唯一硬依赖）
- 可选：fastapi/uvicorn/pydantic（server）、transformers/torch（ml 检测器）、各框架 SDK（集成适配）

### 考古范围声明
- 深度考古面：guard.py / integrity.py / detectors/ / policies/ / events.py / integrations/ / bench/ / scanner/ / mcp-server / examples（secure_langgraph_memory、opentelemetry_hook）/ tests/ / 08-01 以来 210 commits 修复史
- 已知未深读：docs/ 全量、analytics/（下载统计脚本）、templates/、_config.yml（Jekyll 页面）
