# Archon — Knowledge Archaeology 索引

## 一句话定位
coleam00/Archon（TypeScript/MIT/23.7k★）：自托管、**受治理（governed）** 的 agentic automation engine——"The first open-source harness builder for AI coding"；workflow 混合确定性步骤 + AI agents + human gates + audit trails。本考古锁定 **控制面治理**：CVE-2026-111322（0.2.13–0.11.1 `/api/*` 无认证 + 默认绑 0.0.0.0 → RCE，CVSS 9.8）与 0.12.1 修复层形成"事故前 → 修复后"完整对照。

## 产物清单
| 层 | 文件 | 内容 |
|---|---|---|
| Overview | 00_overview.md | 定位 / 认知主线 / 关键文件锚 / 跨项目连接 |
| Project Layer | 01_project-layer/01_project_map.md | 架构 / 生命周期 / 模块 / 治理机制 10 条 / 配置 |
| Engineering Knowledge | 02_engineering-knowledge/ek-graph.md | EK-B01~B21（EK Graph 六类边，含 R-3 新增） |
| Knowledge Layer | 03_knowledge-layer/ko.md | KO-B01~B05（aggregation_rule R1-R4） |
| Flow Atlas | 04_flow-atlas/flow-atlas.md | 七类流 |
| Candidates | 05_candidates/candidates.md | C-B01~B07（Hypothesis 标注） |
| Validation | 06_validation/ | validation + independent-validation-report + reconciliation |
| Runs | archaeology-runs/ARCH-2026-10-21-001/ | job-manifest / snapshot / run_metadata |

## 考古记录
- **ARCH-2026-10-21-001**（initial, v0.12.1 @ 11aba36）— 完整六阶段；Harness/治理 周 Day3（CVE 事件驱动选型）

## 关键发现（3 条）
1. **控制面认证 = opt-in gate + fail-closed 默认**：OFF by default（DATABASE_URL+BETTER_AUTH_SECRET 同现）；弱密钥 boot 拒绝；API gate 默认 401；signup 默认 disabled
2. **live 凭据端点 fail-fast**：/internal/git-credential 分发 live tokens——GitHub App + 非 loopback → **serve 前 fatal**（"WARN 太容易滚动过去"）；显式 opt-out 才放行
3. **身份 = 所有权根**：401/503 严格分离；非 owner 与缺失行同 404（防枚举）；per-user 无凭据 refuse（不借 host 登录）

## 关键争议
- C-B01：0.12.x 治理层是否 CVE-2026-111322 驱动的重建（Hypothesis——0.12.0/0.12.1 同日 10-17 发布，需 git blame）
- EK-B21：v0.12.0 曾被 provider 加载目标仓库 .env 撤销 strip → 0.12.1 修复（#4426/4427）
- 隔离分层：in-place 默认（诚实声明无隔离）vs Box/MXC 最强默认——两种"默认安全"策略的取舍
