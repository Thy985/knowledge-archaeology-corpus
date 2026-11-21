# 01 · Project Layer — Temporal（ARCH-2026-11-02-001）

## 1. 项目定位

- **名称/角色**：Temporal Server——durable execution 引擎的服务端实现（workflow/activity 的持久化调度与状态管理）；客户端 SDK 在独立仓库。
- **生命周期**：2019 年从 Uber Cadence fork 独立；当前 Server 1.32.x（2026-10-09 发布 1.32.1）；本考古 HEAD 2026-10-31。
- **形态**：Go 微服务集合（可单进程多服务合一启动，`temporal/server.go` DefaultServices = frontend+history+matching+worker）。

## 2. 架构（四服务 + Chasm）

```
                 ┌──────────────────────────────────────────────┐
                 │              Frontend Service                │
                 │  gRPC + HTTP API server（对外 API 网关）      │
                 │  admin_handler / namespace / cluster 校验     │
                 └──────┬─────────────────────┬────────────────┘
                        │                     │
               ┌────────▼─────────┐   ┌───────▼────────┐
               │   History Service │   │  Matching Service│
               │ 核心状态机引擎     │   │ 任务队列/分派    │
               │ · Mutable State   │   │ poll 长连接      │
               │ · Event History   │   │ backlog 管理     │
               │ · Workflow Task   │   │                 │
               │ · 队列处理器       │   └───────┬────────┘
               │ · Shard/复制/归档  │           │ AddWorkflowTask
               └──────┬────────────┘           │
                      │                        ▼
               ┌──────▼─────────────┐  ┌───────────────────┐
               │  Persistence Layer │  │   Worker Service   │
               │ execution_manager  │  │ replicator/batcher │
               │ sql/cassandra/     │  │ dlq/parentclosepolicy
               │ nosql + visibility │  └───────────────────┘
               └────────────────────┘
   Chasm（新引擎）：chasm/ ASM 框架（Engine/Tree/Registry/Library），
   History service 内 chasm_engine.go 承载，Workflow 降格为其中一个 ASM
```

- 服务间通信：`client/`（frontend/history/matching/worker 的 client 库）+ gRPC；DI 用 uber-go/fx（各 service fx.go）。
- 依赖注入与装配：`temporal/fx.go` TopLevelModule 组装。

## 3. 核心模块清单

| 模块 | 路径 | 行数 | 职责要点 |
|---|---|---|---|
| Frontend | service/frontend/ | 39,997 | API 网关、admin handler、http_api_server、health_check |
| History | service/history/ | 265,806 | **引擎核心**：api/（per-API handler）、workflow/（MutableState、状态机）、queues/（队列处理器）、shard/、replication/、ndc/（多数据中心）、archival/、chasm_engine.go |
| Matching | service/matching/ | 52,830 | TaskQueue 匹配、poll 管理、backlog、版本化任务队列 |
| Worker | service/worker/ | 49,069 | replicator、batcher、dlq、migration、pernamespaceworker |
| Chasm | chasm/ | 10,2658 | **新 ASM 引擎**：Engine/Tree/Registry/Library/Component/Field/Transition/statemachine + lib/{workflow,scheduler,nexusoperation,callback,namespacereplication} |
| Persistence | common/persistence/ | — | execution_manager.go（Create/Update/ConflictResolve）、sql/cassandra/nosql、visibility |
| API/Proto | api/ + proto/internal/ | 101,047（生成） | gRPC 定义 + persistence protobuf 模型 |
| 公共库 | common/ | — | dynamicconfig、namespace、nexus、auth、authorization、worker_versioning、searchattribute |

## 4. 核心数据结构与状态模型（详见 02 EK-01~EK-12）

1. **Mutable State**（`service/history/workflow/mutable_state_impl.go`，10,308 行）——每 workflow execution 的状态容器：pending activity/timer/signal/child、Event History 当前指针、VersionedTransition、HSM 树 + ChasmTree。
2. **Event History**——不可变追加事件日志；分支通过 branch token 管理（`common/persistence/history_branch_util.go`）。
3. **VersionedTransition**（`state_transition_history.go`）——`{NamespaceFailoverVersion, TransitionCount}`，版本化状态计数，幂等与并发控制基石。
4. **Workflow Task 状态机**（`workflow_task_state_machine.go`，1,608 行）——Scheduled→Started→Completed/Failed/Timeout + transient/speculative 变体。
5. **HSM / ChasmTree**——状态机树；Chasm 用 Node/valueState/Transition 持久化。
6. **Shard / RangeID**——分片所有权 CAS（`common/persistence/data_interfaces.go` ShardOwnershipLostError）。

## 5. 核心执行模型（Replay-based）

详见 04 Flow Atlas 控制流 F1-F8。单行版：
**用户 app → Frontend → History（初始化 History+Mutable State+transfer queue）→ Queue Processor → Matching → Worker 确定性执行 → Command 回传 → History 校验+追加 Event+持久化 → 崩溃恢复 = 从 Event History 重放**。

## 6. 测试体系

- 3,142 个 .go 文件含 1,072 个 *_test.go（34%）；核心包均有测试。
- 集成测试 `tests/`：activity_api_*、admin_*、versioning_*、chasm_test.go、callbacks_test.go、cancel_workflow_test.go 等数十组。
- 构建/质量门：`make lint-code`（golangci-lint）、`make lint-code-fast`、`make proto`（buf+protoc+protoc-gen-go-chasm）、测试须 `-tags test_dep`（集成加 `integration`）。
- test server：`temporaltest/` 提供测试用 server 支持库。

## 7. 配置体系

- `config/development*.yaml` 系列模板（cass/mysql8/postgres12/sqlite/es/s3/dual-es/jwt/集群 A/B/C）+ `docker/config_template.yaml`。
- `common/dynamicconfig/`：运行时热改配置库（Client 轮询变化 + 约束值匹配 findMatch + 缓存；Key 类型含 Namespace/Global 级设置，如 `EnableChasm`、`EnableChasmRequestIDDataNode` 等 3,500+ 行常量）。
- `schema/`：cassandra/mysql/postgresql/sqlite/elasticsearch schema + 版本迁移管理。

## 8. 权限与治理机制

- **自治文档**：`AGENTS.md`（Agent 协作规范：先看代码再改、改 proto 须重新生成、测试须带 tag、make lint 门）+ `.github/copilot-instructions.md`（代码指南）+ CONTRIBUTING.md。
- **服务认证/授权**：`common/auth` + `common/authorization`（namespace 级授权；config/development-jwt.yaml JWT 配置）。
- **错误语义**：`common/serviceerror`（统一 gRPC 错误）；`common/softassert`（可降级断言）。
- **CI**：`.github/workflows/` + `.goreleaser.yml` + `Makefile`（bins/proto/lint/test 全链）。

## 9. 外部依赖（go.mod）

- 存储：cloud.google.com/go/storage、aws-sdk-go-v2（S3）、gocql、go-sql-driver/mysql、pgx、modernc sqlite
- 基础设施：grpc、protobuf、prometheus、opentelemetry、uber-go/fx、zap、bits-and-blooms/bloom
- 子模块：`.gitmodules`（外部 repo 引用）

## 10. 项目状态（本轮视角）

- **活跃开发**：Chasm 引擎（10.2 万行）与 Workflow 并存演进，动态配置大量 Chasm 开关（EnableChasm / ChasmSkipPersistenceEnabled / ChasmSignalBacklinksEnabled 等）→ 处于**双引擎过渡期**。
- **风险点（观察）**：双执行模型（MutableState+HSM vs ChasmTree+Node）并行增加历史服务复杂度；speculative workflow task（优化）与 transient workflow task（复制）并存的状态机分支多——是 failure/边界高发区（见 02 EK-22/23、05 Candidates）。
