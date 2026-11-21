# 00 Overview — Archon（ARCH-2026-10-21-001）

## 定位
Archon（coleam00/Archon，TypeScript/MIT/23.7k★）—— "The first open-source harness builder for AI coding"：自托管、**受治理（governed）** 的 agentic automation engine。Workflow 混合确定性步骤 + AI agents + **human gates** + audit trails；"Make AI coding deterministic and repeatable"。考古当日 HEAD 11aba36（dev，v0.12.1），2026-10-20 仍活跃。

## 为什么它值得考古（本周末日焦点）
- **控制面安全反例 → 修复实证**：CVE-2026-111322（0.2.13–0.11.1 `/api/*` 无认证 + 默认绑 0.0.0.0 → 未授权 RCE，CVSS 9.8，Day3 雷达并入）与当前 0.12.1 的治理层形成**"事故前 → 修复后"完整对照**——比单一防御侧案例（Box/MXC）多出"配置缺失如何导致失控"的攻击侧认知
- **与本周已验证案例构成治理对照对**：Strands Box（OS 遏制，10-20）/ MXC（OS 隔离，10-19）/ agent-governance-toolkit（无状态策略引擎）vs Archon（应用层治理 + 控制面配置）——同一周可升维"控制面安全"认知模型
- Harness 本体（领域中心度）：本周即 Harness/治理 周，Archon 是对研究对象本体的直接考古

## 四条认知主线
1. **控制面认证 = opt-in gate + fail-closed 默认**：web auth OFF by default（需 DATABASE_URL + BETTER_AUTH_SECRET 同现）；弱密钥（<32）boot 拒绝；API gate 默认 401；signup 默认 disabled——"enabling web auth without an allowlist never silently exposes open public registration"
2. **live 凭据端点 fail-fast**：`/internal/git-credential` 分发 live installation tokens；GitHub App + 非 loopback 绑定 → **serve 前 fatal throw**（"a WARN line in startup logs is too easy to scroll past"）；显式 `ARCHON_ALLOW_INTERNAL_ON_PUBLIC_BIND=1` 才 opt-out
3. **身份 = 所有权根**：401（缺失身份）/503（后端故障）严格分离；非 owner 与缺失行同 404（防枚举）；write 归 owner；per-user runs without credentials refuse（不借 host 登录）
4. **隔离 = 显式选择的分层**：in-place（默认，宿主直跑）/ container（opt-in，只读 bind + overlay，docker exec，Phase C 前 diff 丢弃）/ git worktree（"not an execution or security sandbox" 明示）

## 关键文件锚（证据索引）
| 文件 | 内容 |
|---|---|
| packages/server/src/auth/config.ts | 认证 gate 纯函数（enabled/secret/api-gate/signup/header） |
| packages/server/src/routes/api-request-auth.ts | 401/503 语义 + 非 owner 404 + run actor |
| packages/server/src/index.ts:225-545 | assertInternalEndpointBind（fatal）+ warnWebAuthHeaderTrust |
| packages/isolation/src/backends/{in-place,container}.ts | 隔离后端分层 |
| packages/core/src/credentials/delivery.ts | vendor-canonical env-only 凭据交付 |
| AGENTS.md | governed engine 边界 + 工程准则（fail clearly / 机器可查规则） |
| CHANGELOG.md（#3831/4031/4075/4154/4224/4282） | 治理迭代证据 |

## 跨项目连接
- **Strands Box**（10-20）：OS 级外部遏制（防御侧）↔ Archon 应用层治理（进程内）——"agent 需要外部遏制"动机链的治理补充
- **MXC**（10-19）：OS 策略隔离层 ↔ Archon 应用层控制面配置
- **agent-governance-toolkit**：服务端无状态策略引擎 ↔ Archon 本地 governed engine
- **deepseek-harness**：sandbox/approval seam ↔ Archon human gates + audit trails
- **openclaw**（已考古）：同为 agent 运行时，控制面暴露面对照
