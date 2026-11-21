# 01 · Project Layer — Config / Testing / Governance

## 1. 配置体系

### 1.1 配置分层

| 层 | 载体 | 内容 |
|---|---|---|
| 文件 | `ov.conf`（`config/config_file.py`） | server、storage、model、vector、auth、encryption |
| 账户级 | `AccountConfig`（`config/account_config.py:142`） | memory / parser_api / feishu / VLM / vector / ACL / TTL |
| 运行时 | `config/manager.py`（`AccountConfigView`） | 账户视图 + 覆盖合并 |
| 默认值 | `config/ttl.py`、`config/vector.py`、`config/vlm.py` | 阈值与超时 |

关键字段：

| 配置 | 默认 | 影响 |
|---|---|---|
| `server.auth_mode` | — | `api_key` / `trusted` / `dev` / `ldap` / `oidc` |
| `server.root_api_key` | — | trusted 模式的**强制**共享密钥 |
| `acl.enabled` | **false** | 关闭时 ACL 字段照写，但不做访问检查/检索过滤 |
| `encryption.enabled` | **false** | 文件静态加密（`crypto/config.py:35`） |
| `session.agent_evolution_enabled` | 由账户配置 | 关闭时剔除 `cases/trajectories` 记忆类型 |
| `strict_extract_errors` | **false** | 关闭时抽取异常 → warning + 空结果 |

### 1.2 配置合并

`config/merge.py` + `config/scope.py`：账户级覆盖全局；`merge` 对嵌套 dict 合并而非整段替换（代码存在，未逐行验证优先级顺序——见 Candidates C-06）。

## 2. 测试体系

| 项 | 值 |
|---|---|
| 测试文件 | **803** 个 `tests/**/test_*.py` |
| 顶层测试目录 | 33（`session` / `storage` / `retrieve` / `transaction` / `server` / `service` / `vectordb` / `acl` 分散） |
| 覆盖对象 | 单元 + 集成 + API + gateway E2E + CLI + benchmark + 性能 |

测试目录与设计文档的对应关系（抽样）：

| 测试目录 | 对应设计 |
|---|---|
| `tests/transaction/` | `09-transaction.md`：锁、mv、rm、冲突、幂等 |
| `tests/storage/` | VikingFS / 路径 / 队列 |
| `tests/retrieve/` | 意图分析、召回、tier 组装 |
| `tests/session/` | 压缩、抽取、更新、隔离 |
| `tests/vectordb/` | 向量库适配与本地记录格式 |
| `tests/gateway/` | 网关多实例 / E2E |

> 未执行任何测试（只读约束 + 不装依赖）。**测试存在 ≠ 行为已验证**；本 Package 中所有"测试揭示的行为"只作为设计意图的间接证据（S2/S3），不标 S4。

## 3. CI / 质量门

`.github/workflows/` 共 **37** 个 workflow。

| workflow | 触发 | 作用 |
|---|---|---|
| `ci.yml`（02. Main Branch Checks） | push `main`（忽略 docs/md/图片） | CodeQL 安全扫描 |
| `pr.yml` | PR | PR 检查（含 lint / 测试路由） |
| `_test_lite.yml`（12. Lite Test Suite） | 被调用 / 手动 | ubuntu-24.04、Python 3.10 最小矩阵 |
| `_test_full.yml` | 被调用 | 多 OS / 多 Python 全矩阵（含 macOS brew cmake、Windows mingw） |
| `api_test.yml` / `api_test_effect.yml` | — | API 契约测试 |
| `gateway-e2e.yml` | — | 网关端到端 |
| `host-compat.yml` | — | 宿主兼容性 |
| `rust-cli.yml` | — | Rust CLI 构建/测试 |
| `_codeql.yml` | 被 ci 调用 | 静态安全分析 |

**观察**：lite 矩阵只有 Python 3.10 一个版本，说明发布面"最低版本即主测版本"；full 矩阵承担多版本兼容。

## 4. 治理与许可

| 项 | 内容 |
|---|---|
| License | **AGPL-3.0**（`LICENSE` 34,568 B；`pyproject.toml:19`） |
| 版权头 | 源文件统一 `Copyright (c) 2026 Beijing Volcano Engine Technology Co., Ltd.` + `SPDX-License-Identifier: AGPL-3.0` |
| 贡献 | `CONTRIBUTING.md`（15,306 B） |
| 安全 | `SECURITY.md`（1,691 B） |
| PR 自动化 | `.pr_agent.toml`（16,984 B）——PR-Agent 配置 |
| Release | `release.yml` / `release-tos.yml` / 各 SDK release workflow |
| 版本 | setuptools-scm 动态；README 声明 `0.3.22` |

### AGPL 的工程含义（风险，不是知识对象）

- 网络提供服务时，AGPL 要求向用户提供 Corresponding Source；**将 OpenViking 作为 SaaS 后端使用会触发源码提供义务**。
- 内部只读考古不构成分发；但若基于其代码二次开发并对外服务，需法务评估。
- 本轮不复制任何代码进 corpus。

## 5. 版本与发布事实

| 事实 | 证据 |
|---|---|
| `pyproject.toml` 无静态 version，`dynamic = ["version"]` | `pyproject.toml` |
| setuptools-scm 从 git tag 推导 | 同上 |
| 仓库内无 `openviking/_version.py`（try/except 回退 `0.0.0+unknown`） | `openviking/__init__.py` |
| README 写 `openviking 0.3.22` | `README.md` benchmark 段 |
| HEAD 提交是 UI release `#7391` | `git log -1` |

→ **版本结论**：书面版本 0.3.22 无法由代码自证；考古以 commit SHA 为准（`f7e83b2d…044b`），版本号只作 Observation。

## 6. 工程治理的缺口（供 Candidates）

| 缺口 | 事实 | 影响 |
|---|---|---|
| 无 CHANGELOG 强制 | 未见 `CHANGELOG.md` | 版本演进不可追溯 |
| AGPL 无 CLA 说明 | `CONTRIBUTING.md` 未逐行读 | 贡献者许可链未知 |
| 测试未覆盖 Rust 锁内部 | `crates/ragfs` 130k 行，Python 测试只能经 binding | 锁协议正确性依赖 Rust 侧测试 |
| 默认关闭安全能力 | `acl.enabled=false`、`encryption.enabled=false` | 默认部署是"无 ACL、无加密"的 |
