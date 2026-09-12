# Repository Snapshot Artifact — aigis (pyaigis-kr)

## Snapshot

| 字段 | 值 |
|---|---|
| repository | https://github.com/gaebalai/aigis-kr.git |
| commit_sha | `229a2917dfa9ae881ee551c98c3c45a0f43b658d` |
| branch | main（默认） |
| repository_version | `pyaigis-kr` v1.1.4（pyproject `name = "pyaigis-kr"`；pyaigis 的韩国合规 fork 通道） |
| analysis_timestamp | 2026-09-13T02:00+08:00（cron 触发） |
| shallow_clone | `--depth 1` → `archaeology-jobs/ARCH-2026-09-13-001/repo` |
| 规模 | 13MB / 254 .py（主包 aigis/ 93 .py）/ tests 58 文件 |
| GitHub API | language Python、8 stars、archived=False、created 2026-05-18、pushed 2026-05-18T08:23:52Z（**3 个月前停更——诚实记录**） |
| skill_version | knowledge-archaeology v3.2（本地生产版，未修改） |

## 项目基础地图

```
aigis-kr/（pyaigis-kr）
├── aigis/                     # 主包（93 .py）
│   ├── guard.py / scanner.py / policy.py / mcp_scanner.py   # 主入口与扫描器
│   ├── filters/               # L1-L3 检测（fast_screen/input_filter/output_filter/rag_context_filter/patterns/scorer…）
│   ├── safety/                # L4 CaMeL（taint/tokens/enforcer）+ L5 AEP sandbox
│   ├── spec_lang/             # L6/L7 PolicyDSL（parser/evaluator/fsm）+ stdlib
│   ├── audit/                 # HMAC-SHA256 签名日志 + hash chain（signed_log/chain/verify）
│   ├── middleware/            # 适配器：Claude Code hook / FastAPI / LangChain callback / Proxy
│   ├── multi_agent/           # AgentMessageScanner + AgentTopology（信任分级）
│   ├── cross_session/         # SleeperDetector（睡眠注入）+ CrossSessionCorrelator
│   ├── memory/                # 记忆投毒检测/写入过滤
│   ├── policies/              # 合规策略管理（manager）
│   ├── adapters/ aep/ capabilities/ monitor/ supply_chain/
│   └── 顶层：cli.py / compliance.py / compliance_kr.py / compliance_registry.py / redteam.py / adversarial_loop.py / benchmark.py / weekly_report.py / i18n.py
├── tests/                     # 58 文件；实测 1711 passed / 1 failed（LC_ALL 环境耦合）
├── policy_templates/          # 12 YAML 合规模板（含 kr_pipa/kr_isms_p/kr_finance）
├── ARCHITECTURE.md v1.3.1     # 设计意图（4-wall + L1-L7 + MCP + Log 3-tier + Incident）
├── GOVERNANCE.md / SECURITY.md / PENDING_DECISIONS.md / ROADMAP.md / CLAUDE.md
├── backend/ frontend/ sdk/ vscode-extension/ site/ articles/ books/ content/ images/ auto-improvement/
└── pyproject.toml（dependencies = [] 零核心依赖）/ uv.lock / Dockerfile / docker-compose.yml
```

## 核心模块 / 数据结构 / 状态 / 测试 / 配置 / 权限 / 依赖

| 维度 | 事实 | 证据 |
|---|---|---|
| 主要语言 | Python 3.11+（hatchling 打包） | pyproject.toml |
| 入口 | `aig` CLI（guard/scanner/mcp/compliance/redteam/benchmark）+ middleware 适配器 + FastAPI server | aigis/cli.py + middleware/ |
| 核心数据结构 | ActivityEvent（governance flow 载体）、MatchedRule、ScanResult、SignedLogEntry、Rule（Trigger/Predicate/Enforcement）、MCPServerReport | aigis/types.py + audit/signed_log.py + spec_lang/parser.py |
| 核心状态 | Incident：open→investigating→mitigated→closed；Trust Level：trusted/suspicious/dangerous；RunResult（allow/deny exit 0/2） | ARCHITECTURE.md Incident Lifecycle + MCP trust score |
| 测试体系 | pytest 58 文件；**实测 1711 passed / 1 failed**（失败=test_i18n 未 mock LC_ALL，沙箱 LC_ALL=en_US 触发——测试环境耦合，非产品缺陷） | tests/ + 实测 |
| 主要配置 | aigis-policy.yaml（前缀匹配规则→allow/deny）、policy_templates/*.yaml（合规）、AIGIS_LANG/i18n、enterprise_mode（incident 2 层） | policy.py + ARCHITECTURE Governance Flow |
| 权限/治理 | L4 CaMeL capability tokens（file:read/net:connect/exec:shell）+ taint 追踪；审计日志 HMAC + hash chain 防篡改；GOVERNANCE.md lazy consensus；SECURITY.md 72h 响应 | aigis/safety/* + audit/ + GOVERNANCE.md |
| 外部依赖 | **零核心依赖**（dependencies=[]）；可选：pyyaml（策略文件）、FastAPI/PostgreSQL（enterprise incident）、cryptography（Ed25519 可选升级） | pyproject.toml + PENDING_DECISIONS D8 |
