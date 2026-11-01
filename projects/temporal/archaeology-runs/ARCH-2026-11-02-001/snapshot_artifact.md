# Snapshot Artifact — Temporal（ARCH-2026-11-02-001）

## 1. 快照元数据

| 项 | 值 |
|---|---|
| repository | https://github.com/temporalio/temporal.git |
| branch | main（single-branch shallow clone, depth=1） |
| commit SHA | `e0233ff6658fcab7b4e7ecc1c16af40baf629b78` |
| commit 时间 | 2026-10-31 18:08:29 UTC（"Bump go.temporal.io/testx to ddcb07397"） |
| go.mod | `go.temporal.io/server`，**go 1.27.3** |
| 工作区体积 | 64MB（4123 文件检出；仓库 API size=174341KB） |
| 总 Go 代码 | **1,124,761 行**（含 api/proto 生成代码 101,047；业务代码 1,023,714） |
| 服务分解 | history 265,806 / matching 52,830 / worker 49,069 / frontend 39,997 |
| Chasm 框架 | chasm/ 10,2658 行（新 ASM 引擎，独立于 workflow） |

## 2. 项目定位（一句话）

Temporal = **durable execution 的开山鼻祖与工业标准**——一个面向分布式应用/workflow 的持久化执行引擎：workflow 代码以"确定性重放"（Replay-based）执行，所有状态变更以 **Event History**（事件历史）追加写入持久化层，worker 崩溃后从事件流重放恢复，不重放副作用。

## 3. 语言与运行时

- **Go 1.27.3**（go.mod）；TypeScript/Java/Python SDK 在外部仓库（go.temporal.io/sdk 等），本仓库为 Server。
- 二进制入口：`cmd/server/main.go`（temporal-server）；`temporal/server.go` 定义 `Server` 接口与四服务集合（Frontend/History/Matching/Worker，DefaultServices）。

## 4. 核心模块（Architecture）

| 模块 | 路径 | 职责 |
|---|---|---|
| **Frontend** | service/frontend/ | 对外 API 网关（gRPC + HTTP API server），admin_handler、命名空间/集群校验 |
| **History** | service/history/ | **核心状态机引擎**（265,806 行）：Mutable State 管理、Event History 追加、Workflow Task 状态机、队列处理器（transfer/timer/visibility）、Shard 管理、Replication、Archival |
| **Matching** | service/matching/ | 任务队列匹配（Workflow/Activity Task 分派），backlog 管理、poll 长连接 |
| **Worker** | service/worker/ | 服务内 worker：replicator、batcher、deletenamespace、parentclosepolicy、dlq 等 |
| **Chasm** | chasm/ | **新执行引擎**（Coordinated Heterogeneous Application State Machines，102,658 行）：把 Workflow 降格为"一个 ASM"，提供轻量级 typed 状态机框架（Field[T]/Map/ParentPtr + Tasks），复用 sharding/路由/原子存储，支撑 scheduler/nexusoperation 等非 workflow 状态机 |
| **Persistence** | common/persistence/ | 持久化抽象：execution_manager + sql / cassandra / nosql 插件 + visibility（OpenSearch/ES） |
| **API/Proto** | api/ + proto/internal/ | gRPC 服务定义（adminservice/historyservice/matching/…）+ persistence protobuf 模型 |

## 5. 核心数据结构与状态模型

- **Mutable State**（`service/history/workflow/mutable_state_impl.go`，10,308 行）：每个 workflow execution 的**内存+持久化状态容器**——当前 Event History、Pending 命令、Activity 状态、Timer、Signal、ChildWorkflow、VersionedTransition 等。
- **Event History**：不可变追加日志（WorkflowExecutionStarted/WorkflowTaskScheduled/WorkflowTaskCompleted/ActivityTaskScheduled 等事件）；branch token 支持 reset/replay 分支。
- **VersionedTransition**（`state_transition_history.go`）：`{NamespaceFailoverVersion, TransitionCount}` 组合，记录每次状态变更的版本化计数（failover 语义），用于幂等与并发控制。
- **HSM（Hierarchical State Machine）**：mutable state 内部的状态机树（`hsm.Node`），activity/timer/child workflow 各自状态机挂载其上；**ChasmTree** 为 Chasm 新引擎的树。
- **Workflow Task 状态机**（`workflow_task_state_machine.go`，1,608 行）：Scheduled→Started→Completed/Failed/Timeout 的状态转换 + 版本校验（Speculative Workflow Task 优化）。

## 6. 核心执行模型（Replay-based Durable Execution）

1. 用户 app 发 `StartWorkflowExecution` → Frontend → History：初始化 History `[WorkflowExecutionStarted, WorkflowTaskScheduled]` + 持久化 Mutable State + 入 transfer queue。
2. Queue Processor 取出 WorkflowTask → Matching 分派 → Worker 执行 workflow 代码（确定性）。
3. Worker 的每一步副作用（activity/timer/signal）都作为**命令**发回 → History 校验（command_handler）→ **追加 Event 到 History** + 更新 Mutable State + 持久化。
4. 崩溃恢复：新 worker 拿到完整 Event History **重放** workflow 代码，恢复到最后状态；已完成命令在重放中"跳过执行"（已存在于 History），**不重复副作用**。
5. 时间/随机/并发等不确定操作全部走 Temporal 抽象（Workflow.Sleep/Timer/RandomUUID 注入）保证确定性。

## 7. 测试体系

- 单元测试：3,142 个 .go 文件含 1,072 个 *_test.go（34%）；core 包均有 *_test.go。
- 集成/端到端测试：`tests/`（数十个 x_test.go：activity_api_*、admin_*、versioning_*、chasm_test.go 等）。
- 工具：`make lint-code`（golangci-lint）、`make lint-code-fast`、`make proto`（buf + protoc + codegen）、`make test`；测试须带 `-tags test_dep`，集成测试带 `integration` tag。
- temporaltest/：test server 支持库。

## 8. 配置体系

- `config/`：development*.yaml 模板（cass/mysql8/postgres12/sqlite/es/s3/dual-es/jwt/集群 A-B-C），docker.yaml。
- `docker/config_template.yaml`：容器部署模板。
- `common/dynamicconfig/`：**动态配置库**（运行时热改，如 Chasm 开关：`ChasmEnabled`/`ChasmSkipPersistenceEnabled`/`ChasmSignalBacklinksEnabled` 等——见 mutable_state 中大量 Chasm 配置读取）。
- `schema/`：**数据库 schema**（cassandra/mysql/postgresql/sqlite/elasticsearch + version.go 迁移版本管理）。

## 9. 权限与治理机制

- `AGENTS.md`：项目自治文档（Agent 协作规范）——代码审查/修改须遵循 `github/copilot-instructions.md`；明确项目结构、开发命令（make lint/proto/test）、工作流（先读代码再改、改 proto 须重新生成）。
- `CONTRIBUTING.md` + `.github/`（CI/actions）+ `.goreleaser.yml`。
- 服务间认证：`common/auth` + `common/authorization`（namespace 级授权、JWT 配置模板 config/development-jwt.yaml）。
- 错误语义：`common/serviceerror` 统一错误码；`common/namespace` 命名空间缓存与隔离。

## 10. 外部依赖（go.mod 主要）

- 云存储：cloud.google.com/go/storage、aws-sdk-go-v2（S3）、gocql（Cassandra）、go-sql-driver/mysql、pgx、modernc sqlite
- 基础设施：grpc、protobuf、prometheus client、opentelemetry、uber-go/fx（DI 框架，`temporal/fx.go` 组装）、zap
- 工具：Masterminds/sprig、bits-and-blooms/bloom（去重过滤器）等
- 子模块：`.gitmodules` 存在（外部 repo 引用）

## 11. 快照完整性说明

- 浅克隆 depth=1：仅 main 分支 HEAD，无历史 commit（本次考古以"当前实现事实"为证据面，不做 git 考古）。
- Chasm 引擎（102K 行）是本仓库**新近重大方向**（Workflow 之外的第二个执行模型），为工程知识重点采样区。
- 无 Docker 环境：本次为静态源码考古，不跑集成测试；证据强度上限 S4（测试代码揭示的行为）——测试断言仅作参考，不升级为"运行验证"。
