# 01 Project Layer · Config, Testing, Governance

## 1. 配置

| 配置面 | 证据 |
|---|---|
| 包元数据 | `pyproject.toml`: name `letta`, version `0.16.8`, `requires-python >=3.11,<3.14`, Apache-2.0 |
| 数据库 | SQLAlchemy async + Alembic（167 个迁移）；默认 Postgres（pgvector） |
| 向量后端 | `archive.vector_db_provider`（默认 pgvector，可选 `TPUF`）；`insert_passage` 内分派 |
| LLM | `llm_api/` 多 provider；`local_llm/`（ollama / vllm / lmstudio） |
| 鉴权 | `auth_token.py:11 get_current_user`：请求 password 或 api_key → user |
| 记忆上限 | `CORE_MEMORY_BLOCK_CHAR_LIMIT`（`constant`），Block `limit` 默认取该值 |
| Agent 类型 | `agent_type` 影响 prompt 渲染分支（`memory.py:692-703`） |

## 2. 测试体系

- 总量：`tests/` 166 个 `.py`；`test_*.py` 95 个；集成测试以 `integration_test_*` 命名单独成组。
- 与记忆直接相关：
  - `test_memory.py`
  - `test_block_manager_noop_update.py`（无变化更新应不写库/不产生副作用）
  - `test_compaction_thresholds.py`（压缩阈值）
  - `test_static_buffer_summarize.py`
  - `integration_test_summarizer.py`
  - `test_git_http_post_push_sync.py` + `integration_test_git_http_post_push_sync.py`（git 记忆同步）
  - `integration_test_multi_agent.py`（多 agent）
  - `test_multi_agent.py`
- CI：31 个 workflow，含 `core-unit-test.yml`、`core-unit-sqlite-test.yaml`、`core-integration-tests.yml`、`migration-test.yml`、`docker-integration-tests.yaml`、`test-ollama.yml`、`test-vllm.yml`、`test-lmstudio.yml`。

**未验证**：本环境无 DB / 向量库 / LLM key，上述测试一个都未执行；测试揭示的行为只能作为"代码声称的覆盖范围"，不构成 S4 证据。

## 3. 治理与安全面

| 层 | 事实 | 证据 |
|---|---|---|
| 仓库状态 | AGENTS.md 首行："This repository is deprecated." 维护模式，活动开发在 letta-code / Agent SDK | `AGENTS.md` |
| 漏洞流程 | SECURITY.md 只提供邮件上报、48h 确认、7d 修复时间线；**无威胁模型/安全设计说明** | `SECURITY.md` |
| AI 贡献政策 | AI 辅助须披露、人类须理解并复核；禁止自我推广 | `AGENTS.md` + `AI_POLICY.md` |
| 租户隔离 | 所有实体继承 `OrganizationMixin`；block/passage 追加 `ProjectMixin` | `orm/block.py:22` |
| 块级权限 | `read_only` 注释为 "permissions of the agent"，但字段在 block 上 | `orm/block.py:48` |
| 审计 | `BlockHistory.actor_type/actor_id`；`created_by_id/last_updated_by_id` | `orm/block_history.py:26-31` |
| 工具执行 | `core_tool_executor` 在 memory 编辑前检查 `read_only`（至少 9 处） | `core_tool_executor.py:320/336/354/530/665/691/744/899/959` |
| 直接调用保护 | `function_sets/base.py` 的 `memory()`、`memory_replace`、`memory_apply_patch` 均 `raise NotImplementedError("This should never be invoked directly.")` | `base.py:10/311/453` |

## 4. 权限模型的三个事实

1. **隔离的粒度是组织/项目，不是 agent**：agent 之间共享 block 依靠 pivot 表，没有 per-agent 的 block 权限表。
2. **`read_only` 是块级而非关系级**：同一 block 若被 A、B 两个 agent 共享，无法表达"A 可写、B 只读"。
3. **`read_only` 已标记 deprecated**：`BlockResponse.read_only` 带 `deprecated=True`（`schemas/block.py:103`），但 ORM 与 executor 仍在使用——API 契约与内部实现不同步。

## 5. 扩展机制

- 工具集：`function_sets/{base,builtin,files,multi_agent,voice}.py`；工具注册/执行经 `tool_executor` 分派。
- MCP：`routers/v1/mcp_servers.py`；有 `test_mcp_encryption.py`、`test_mcp_url_validation.py`。
- 模板：Block `is_template` / `template_name` / `preserve_on_migration`；human/persona 作为内置模板。
- 多 agent：`multi_agent` 工具集 + `agents/letta_agent_batch.py`；archives 可跨 agent 共享（`archives_agents`）。
