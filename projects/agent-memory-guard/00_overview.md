# 00 Overview — OWASP Agent Memory Guard（ARCH-2026-10-07-001）

## 项目一句话定位
OWASP 官方 Incubator 的**运行时记忆防御层**：插在 AI agent 与其 memory store 之间，用 12 检测器 + 声明式策略（YAML）+ 分类晋升图 + SHA-256 完整性基线，防护 **ASI06 Memory Poisoning**（以及工具滥用/权限提升/过度自主）——"防止 agent 的记忆被毒化，且毒化后的记忆可检测、可隔离、可回滚"。

## 核心命题（本项目让我们认识到什么）
"记忆安全"不是单一的注入检测，而是**分层防线**：内容检测（regex/ML）→ 语义分类（provenance 信任梯度）→ 完整性基线（SHA-256）→ 策略执行（block/redact/quarantine）→ 事件观测（SIEM/OTel）→ 可判定基准（AMSB）。且每一层都诚实声明自己的边界（permissive 默认 / source_class 不认证 / digest 不验证 / 数字附语料）。

## 关键事实速览（全部可追溯仓库实际内容）
- 版本 0.3.3（Release 0.3.3 #152；commit `8403091`，2026-10-06），Apache-2.0，Python ≥3.9，Beta
- 核心依赖仅 `PyYAML>=6.0`（pyproject.toml）；可选中速：server(fastapi/uvicorn/pydantic)、ml(transformers/torch)、langchain/crewai/llamaindex/autogen/openai-agents
- 核心库 51 py（src/agent_memory_guard/）；全仓 120 py、tests 38 文件；210 commits（08-01 以来）
- 下载量：README 声称 14,949 PyPI downloads · 14,174 repository clones（仓库自报）
- 质量指标（compliance-mapping.md 明示，pinned to 0.2.2）：recall 92.5%、precision 100%、FP 0%、F1 0.961、中位延迟 62µs——55 自编 cases（40 attack + 15 benign）；**文档同时声明"不代表对自适应对手的度量"**
- AMSB（Agent Memory Security Benchmark）：21 scenarios（15 malicious + 6 benign），canary 存活判定，benign 反过拦截惩罚，critical breach 封顶 C/D
- 集成面：LangChain（GuardedChatMessageHistory + middleware）、OpenAI Agents（HITL queue）、AutoGen、agno、CrewAI、LlamaIndex、mem0、LangGraph（示例）、OpenTelemetry（hook）、FastAPI/Flask middleware、MCP server、GitHub Action + semgrep + SARIF
- 合规映射：NIST AI RMF 1.0 + EU AI Act（docs/compliance-mapping.md）

## 考古范围
- 深读：guard.py(768L)/integrity.py/events.py/middleware.py/policies/policy.py/classification.py/storage/{memory_store,snapshots}.py + 12 检测器 + bench/{harness,scenarios,scoring,adapter,report,cli} + cli.py + mcp-server + scanner + examples/{secure_langgraph_memory,opentelemetry_hook} + 关键修复 commit（#122/#124/#135/#138/#139/#147/#150/#151）+ tests 38 文件 + docs/compliance-mapping.md + README
- 未深读（声明）：docs/ 其余全量、analytics/、templates/、_config.yml（Jekyll 页面）、CHANGELOG 全史

## 与已有 Corpus 的连接
- **langgraph**（10-06 考古）：`examples/secure_langgraph_memory.py`——StateGraph 节点内插 Guard.write 防护 checkpoint.tool_observation——"记忆三层分工"的防护侧对照
- **owasp-agentic-skills**（09-26 考古）：OWASP www-project 家族；本仓库是 ASI 侧（应用威胁）而非 AST 侧（Skills 威胁）
- **letta-code / mem0**：记忆系统实现（无防护）vs 本仓库（防护中间件）——同一"agent 记忆"域的两个互补视角
- **aigis / guardian / ai-protector**：Agent 运行时防御品类；本仓库补"记忆完整性 + 分类晋升"两个独特维度
- **mcp**：amg-mcp-server（FastMCP 包装扫描器）
- **opentelemetry**：OTel hook（Layer 2 审计轨迹）

## 认知状态声明
- 本包 Fact/Observation/Hypothesis/Pattern/Principle 严格分层（见 03 与 06）；跨项目推广均标注 `Cross-project validation pending`
