# 01.5 Governance & Authority

## 文档定义的权威链

| 文档 | 性质 | 内容 |
|---|---|---|
| `README.md` | 产品定位/能力声明 | "Graphiti = framework；Zep = unified, governed, served context layer" |
| `AGENTS.md` | 仓库操作规则 | 结构、命令、风格、测试、提交/PR 规则；**已知失败与 gotcha** |
| `CONTRIBUTING.md` | 贡献流程 | 开发/测试/PR |
| `CODE_OF_CONDUCT.md` | 社区行为 | — |
| `SECURITY.md` | **安全边界声明** | 参考服务不安全，显式负向清单 |
| `spec/*.md`（7 份） | 设计意图（非实现事实） | driver 重设计、LLM runtime、configurable prompts、C2 默认抽取 prompt、JEV/OpenAI decisions client |
| `signatures/version1/cla.json` | CLA 签名库 | 贡献者授权记录 |
| `LICENSE` | Apache-2.0 | — |

## 安全权威（唯一的显式声明）

`SECURITY.md` 原文要点（**这是仓库自己对"不安全"的权威声明**）：

1. 参考服务 `mcp_server/` 与 `server/` **无认证、无授权**；
2. **无 prompt-injection 防护**：episode 内容与工具入参不经过滤直接送 LLM；
3. **不隔离调用方**：任何 caller 可读/改/删任何 `group_id` 的数据；
4. 示例 Docker Compose 使用默认数据库凭据；
5. **Graphiti 对同一 `group_id` 内所有 episode 给同等信任**：新 episode 可改变/失效旧事实；
6. 上述项被明确列为 **out of scope** 的漏洞报告类型（即"设计如此"）。

**含义**：开源 Graphiti 的信任边界只到 `group_id`，**不到 episode 来源、不到调用者身份**。任何"记忆投毒"防护必须由上层应用实现（选择哪些内容入库、入哪个 group）。

## 权限模型（开源侧实际）

| 控制 | 实现 |
|---|---|
| 分区隔离 | `group_id` 逻辑分区；FalkorDB 物理分库 |
| 跨区查询 | 显式 `group_ids`；decorator 逐区扇出后 merge |
| 认证 | ❌ 无 |
| 授权（ABAC/RBAC） | ❌ 无 |
| 审计 | ❌ 无业务审计；仅 OTEL span/trace |
| 密钥管理 | ❌ 环境变量注入，示例默认凭据 |

## 治理/发布

- **CLA 门禁**：`cla.yml` + `signatures/version1/cla.json`；外部贡献需签 CLA。
- **发布流水线**：`release-graphiti-core` / `release-mcp-server` / `release-server-container` 三条独立 release workflow。
- **Issue/PR 自动化**：`issue-intake`、`pr-intake`、`stale`、`ai-moderator`。
- **依赖治理**：`uv.lock` 锁定；`codeql` 安全扫描。
- **弃用治理**：Kuzu extra 注释"上游不再维护，未来移除"（`pyproject.toml`），但代码仍保留驱动。

## 设计意图 vs 实现（显式分层）

- `spec/driver-operations-redesign.md` 的目标架构（Namespace→Operations ABC→Driver）**已部分落地**（`namespaces/` 733 行存在），但状态标注 **Draft (in progress)**，且 Phase 1 保持非破坏（旧方法仍在数据类上）。→ 不能把 spec 写成"已完成重构"。
- `spec/llm-runtime.md`（2026-08-07 起草）描述的 `LLMRuntime` 已在 `llm_client/llm_runtime.py` 实现，并有对应测试（`tests/llm_client/test_llm_runtime.py`）。→ 属"设计→实现"已闭环的一例。
