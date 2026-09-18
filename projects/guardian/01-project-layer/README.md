# 01 — Project Layer（guardian / legionforge-guardian 项目地图）

## 1.1 它是什么 / 怎么运行

- **定位**：确定性安全 sidecar，插在任意 LLM Agent 框架与工具执行之间；工具调用前跑 7 项检查，fail-fast，无 LLM 调用。
- **运行入口**：`uvicorn legionforge_guardian.app:app`（:9766）；端点 /check（同步热路径）/health/metrics（Prometheus）/rules（查询+invalidate-cache）；客户端 sdk/client.py。
- **部署**：docker-compose（+自带 PostgreSQL，init.sql 自动跑）或独立部署（GUARDIAN_DB_URL 指向已有 LegionForge DB——init.sql 注释：伴生部署时不跑本文件）。

## 1.2 核心模块

```
app.py                全部核心（1412 行）：7 项检查 + JWT 验证 + 缓存刷新 + 审计链 + canary + 指标 + 端点
sdk/client.py         客户端 SDK（176 行）
checks/               __init__ 占位（检查实现内联 app.py——小项目单文件承载）
init.sql              tool_registry / threat_rules / agent_profiles / threat_events / audit_log + canary seed
zap.yaml              OWASP ZAP DAST（:9766，failOnError: true）
.github/workflows/    ci / dast / lint-workflows / oss-audit / publish / test
```

## 1.3 生命周期（工具调用治理决策）

```
Agent 调用工具（tool_id + args + action + task_token）
  → sidecar /check 收到 GuardianCheckRequest
  → Bearer auth 校验（GUARDIAN_REQUIRE_AUTH=true 时；misconfig/auth 失败 → halt fail-safe）
  → 缓存刷新（_maybe_refresh_caches）
  → Check 0 Task token ACL（JWT 有效 + granted_tools 内）
  → Check 1 Tool registry（revoked 优先，10s 传播）
  → Canary 检查（guardian_canary 调用 = 探测证据 → halt + CANARY_TRIGGERED）
  → Check 2 Capability boundary（action 类型 + tool_id 名称双路径）
  → Check 3 Destructive pattern（9 regex 族 → halt / log-only 分级）
  → Check 4 Sequence contracts（playbook 前缀匹配 → novel sandbox）
  → Check 5 Hash integrity（description/schema hash 篡改检测）
  → Check 6 Adaptive rules（DB 60s 热加载：block/injection/sequence）
  → 允许 / halt / sandbox / log-only + threat_events + audit_log hash chain 记录
```

## 1.4 配置

| 配置 | 说明 | 证据 |
|---|---|---|
| GUARDIAN_REQUIRE_AUTH | Bearer auth 强制；misconfig（TASK_TOKEN_SECRET 未设）→ halt | app.py:418-459 |
| TASK_TOKEN_SECRET / TASK_TOKEN_ISSUER | JWT 签名密钥/发行者 | app.py:287 + _GUARDIAN_TOKEN_ISSUER |
| _CACHE_TTL_SECONDS | 注册表/撤销传播延迟（10s） | app.py:352 cache loop |
| adaptive rules | threat_rules 表 60s 热加载 | app.py:482-574 |
| GUARDIAN_DB_URL / GUARDIAN_DB_PASSWORD | PostgreSQL 连接 | app.py:467 + docker-compose |

## 1.5 权限与治理

- **7 项确定性检查**：ACL（JWT scope）/ 注册表（revoked 优先）/ 能力边界（forbidden 名单）/ 破坏性模式（9 regex 族）/ 序列契约（playbook）/ 哈希完整性（篡改检测）/ 自适应规则（热加载）。
- **审计链**：audit_log 表 SHA-256 hash chain（prev_hash + genesis + row_hash 后更新）；**写失败非致命**（"must never block the action it's auditing"）。
- **治理（SECURITY.md）**：OWASP SAMM；GitHub Action 全部固定 SHA；dependabot/renovate 自动更新 + 人工审查；漏洞私有报告（email jp@legionforge.org）。
- **CI/CD**：6 workflows 含 DAST（ZAP 主动扫描，failOnError）、oss-audit、zizmor（workflow 安全静态分析）。

## 1.6 外部依赖

- **非零依赖**：fastapi/uvicorn/pydantic/psycopg/httpx（sidecar + DB 架构——与 Aigis 零核心依赖形成品类内对照）。
- 测试依赖：respx（SDK mock）。
