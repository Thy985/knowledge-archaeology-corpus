# Repository Snapshot Artifact — guardian（legionforge-guardian）

## Snapshot

| 字段 | 值 |
|---|---|
| repository | https://github.com/LegionForge/guardian.git |
| commit_sha | `75c1aaba928d8b1a9b32266081487e4863215f9c`（Merge PR #54 from dependabot，main） |
| branch | main |
| repository_version | pyproject `legionforge-guardian` v0.1.1（PyPI alpha；README quickstart 显示服务 version "4.0.0"——版本号不一致，见 C-04） |
| analysis_timestamp | 2026-09-14T02:00+08:00（cron 触发） |
| shallow_clone | `--depth 1` → `archaeology-jobs/ARCH-2026-09-14-001/repo` |
| 规模 | 256K / 10 .py（app.py 1412 行 + sdk/client.py 176 行 + checks/__init__） |
| GitHub API | 限流未取（core 0/60）——仓库存在性经 git ls-remote 确认（HEAD 75c1aab，main） |
| skill_version | knowledge-archaeology v3.2（本地生产版，未修改） |

## 项目基础地图

```
LegionForge-Guardian/
├── src/legionforge_guardian/
│   ├── app.py                  # 全部核心：FastAPI sidecar + 7 项检查 + 审计链 + canary + 指标（1412 行）
│   ├── checks/                 # 命名空间占位（检查实现内联于 app.py）
│   ├── sdk/client.py           # 客户端 SDK（176 行）
│   └── __main__.py / __init__.py
├── tests/                      # test_checks(603) / test_live(156) / test_sdk(208) = 967 行
│                               # 实测：57 passed / 9 skipped（live 需 GUARDIAN_TEST_URL）
├── init.sql                    # PostgreSQL schema（tool_registry / threat_rules / agent_profiles / threat_events / audit_log + canary seed）
├── zap.yaml                    # OWASP ZAP DAST 配置（localhost:9766，failOnError）
├── docker-compose.yml / Dockerfile / .env.example
├── .github/workflows/          # 6 个：ci / dast / lint-workflows / oss-audit / publish / test
├── .github/dependabot.yml / renovate.json / zizmor.yml / gitleaks.toml
├── SECURITY.md                 # OWASP SAMM + Action SHA 固定 + 私有报告
├── CHANGELOG.md / CLAUDE.md / Makefile
└── pyproject.toml              # fastapi/uvicorn/pydantic/psycopg/httpx 依赖（非零依赖）
```

## 核心模块 / 数据结构 / 状态 / 测试 / 配置 / 权限 / 依赖

| 维度 | 事实 | 证据 |
|---|---|---|
| 主要语言 | Python 3.11+（setuptools 打包） | pyproject.toml |
| 入口 | FastAPI sidecar :9766（/check 同步热路径）+ sdk/client.py | app.py:1194 check + sdk |
| 核心数据结构 | GuardianCheckRequest/Response（tool_id/args/action/task_token/agent_id/run_id）、GuardianTaskToken（JWT granted_tools）、audit_log 行（seq/ts/event_type/payload/prev_hash/row_hash） | app.py:583-611 + init.sql |
| 核心状态 | tool_registry status（APPROVED/REVOKED/PENDING）；threat_rules status（APPROVED/PENDING/REJECTED/EXPIRED）；响应 tier（halt/sandbox/log-only） | init.sql + check 3 实现 |
| 测试体系 | pytest 967 行；**实测 57 passed / 9 skipped**（live 需真实服务；单元测试全绿） | tests/ + 实测 |
| 主要配置 | GUARDIAN_REQUIRE_AUTH（Bearer 强制）、TASK_TOKEN_SECRET/ISSUER、_CACHE_TTL_SECONDS（10s 传播）、adaptive rules 60s 热加载 | app.py:418/352 + check 6 |
| 权限/治理 | 7 项检查（Task token ACL / registry / capability / destructive / sequence / hash / adaptive）；hash chain 审计；SECURITY.md SAMM + Action 固定 + dependabot | app.py + SECURITY.md |
| 外部依赖 | fastapi/uvicorn/pydantic/psycopg/httpx（**非零依赖**——与 Aigis 零依赖对照）+ 可选 respx（测试） | pyproject.toml |
