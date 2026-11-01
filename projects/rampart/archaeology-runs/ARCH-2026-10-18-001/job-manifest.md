# Job Manifest — ARCH-2026-10-18-001

> run_id: `ARCH-2026-10-18-001` ｜ 日期: 2026-10-18（Sunday）｜ 领域: Evaluation 周 Day7 收官 + 换域 Harness/治理（10-19 起）

## 候选排序（按 Knowledge Value / Agent-AI 相关性 / Novelty / 增长信号 / 与 Corpus 连接，不按 star）

| 排名 | 候选 | 状态 | 理由 |
|---|---|---|---|
| P0 | **RAMPART**（microsoft/RAMPART） | **REFRESH_REQUIRED** | 09-04 考古后 6 周新增 7 个 feature 级 commit；Harness/治理 领域核心锚点（agent 安全测试框架）；与已考古 PyRIT 同生态；与 silver-shield/E2E-CLI 连接 |
| P1 | OpenCode（opencode-ai/opencode） | COMPLETED | 09-09 已考古；仓库已归档（迁移 charmbracelet/crush），无 refresh 价值 |
| P2 | MXC（microsoft-execution-containers） | QUEUED（无公开仓库） | 候选卡存在但官方仓库未公开，不可考古 |
| P3 | OWASP Agentic Skills | COMPLETED | 已考古（corpus 有 owasp-agentic-skills） |
| P4 | Evaluation 周其余候选（AgentJudgeBench/Transect/求索 4.0/TestJack/LiveMACE/ToFu/PI-Hunter） | REJECTED | 均无官方公开仓库（10-17 轮已核验排除） |

## 选中项目

- **project**: rampart
- **repository**: https://github.com/microsoft/RAMPART.git
- **mode**: **refresh**（上次考古 ARCH-2026-09-04-001，基线 commit `125595cb2dc53ed81ac4da0f30a7042d6951e897`）
- **priority**: P0
- **reason**:
  1. 明确 refresh 信号：09-04 → 10-18 间新增 [FEAT] onboarding skill(#195)、durable terminal evaluation contract(#146)、canonical versioned trace/result serializer(#185)、configurable trial populations(#123)、shared linear trace runner(#148) 等 7 个 feature 级 commit（10-15 仍在 push）
  2. 领域衔接：明天（10-19）起进入 Harness/治理 周——RAMPART 正是该领域核心锚点（pytest-native agent 安全测试 = 治理落地机制），今日 refresh 为换域提供新鲜事实底座
  3. Corpus 连接：RAMPART 建立在已考古的 PyRIT 之上（corpus 有 pyrit/），同生态对照；与 agent-governance-toolkit（已考古）互补
  4. 用户项目连接：silver-shield（防诈骗=AI 安全应用）、E2E-CLI（测试方法论同构）
  5. Benchmark 学习价值：pytest-native 安全测试框架 + CI/CD 内嵌——对引擎 evaluation 层"把安全测试当 CI 测试写"有直接参照
- **source_entry**: `03_expansion_queue/candidates/[cand]rampart-clarity-2026.md`

## 本次 refresh 重点（对比 09-04 基线）

1. 新增组件：RAMPART onboarding skill（#195）
2. 评测契约：durable terminal evaluation contract（#146）——terminal 状态的可持久化评测契约
3. 序列化：canonical versioned trace/result serializer（#185）——trace/result 版本化规范序列化
4. 试验配置：trial populations 替代 trial cloning（#123）
5. 共享线性 trace runner（#148）
6. 修复：duplicate timestamps 报告保留（#179）
7. 09-04 基线已验证的核心机制是否变化（trial/terminal/event/red-teaming API 面）
