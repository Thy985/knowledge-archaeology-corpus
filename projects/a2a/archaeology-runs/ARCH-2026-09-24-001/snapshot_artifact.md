# Repository Snapshot — A2A（Agent2Agent Protocol）

| 字段 | 值 |
|---|---|
| run_id | ARCH-2026-09-24-001 |
| repository | https://github.com/a2aproject/A2A.git |
| commit SHA | `43e0c874d3baba68ed84b98678d7f2268438e69f` |
| branch | main（HEAD） |
| commit 消息 | `docs: add MolTrust to partners (#2252)` |
| commit 时间 | 2026-09-22 15:27:09 +0200 |
| snapshot timestamp | 2026-09-24（本 run） |
| repository version | A2A Protocol v1.0.1（CHANGELOG 最新发布；README 徽章 pypi a2a-sdk） |
| 协议版本声明 | AgentCard `protocol_version` 支持 "0.3" / "1.0"（a2a.proto:355） |
| 大小 | 135 tracked files；核心规范 docs/specification.md 3618 行 |

## 项目基础地图

```
a2aproject/A2A
├── specification/            # 协议定义（唯一"可执行资产"）
│   ├── a2a.proto             # 812 行 Protocol Buffers 定义（service + 28 message + 3 enum）
│   ├── buf.yaml / buf.lock / buf.gen.yaml   # protobuf 工具链
│   └── json/README.md        # JSON 模式说明
├── docs/
│   ├── specification.md      # 3618 行正式规范（normative，RFC 2119 语言）
│   ├── topics/               # 11 个主题文档（life-of-a-task / agent-discovery / streaming / extensions / governance / multi-tenancy / a2a-and-mcp / …）
│   ├── blog/  tutorials/  sdk/  roadmap.md  partners.md  whats-new-v1.md
│   └── llms.txt              # LLM 摘要
├── adrs/
│   ├── adr-001-protojson-serialization.md   # 唯一 ADR（已 Accepted）
│   └── adr-template.md
├── scripts/                  # 构建/格式/模式生成（proto_to_json_schema.sh 等 8 个）
├── .github/
│   ├── workflows/            # 11 个 CI（lint/spelling/release-please/docs/conventional-commits/issue-metrics/stale/links/linter）
│   └── actions/ + CODEOWNERS + PULL_REQUEST_TEMPLATE
├── GOVERNANCE.md             # TSC 八席位治理宪章（Google/MS/Cisco/AWS/Salesforce/ServiceNow/SAP/IBM）
├── SECURITY.md               # GitHub Security Advisories 报告路径
├── MAINTAINERS.md / CONTRIBUTING.md
└── CHANGELOG.md              # v0.2.1 (2025-05-27) → v1.0.0 (2026-03-12) → v1.0.1 (2026-05-26)
```

## 识别结果

### 主要语言 / 技术形态
- **Protocol Buffers（proto3）**——协议定义语言（`specification/a2a.proto`）
- 非传统代码库：仓库是**协议规范 + 文档 + 治理**资产；实现（SDK）在独立仓库（a2a-sdk 等，README 徽章指向 PyPI）

### 主要运行入口
- **无服务器运行时**（规范库）。逻辑入口 = `specification/a2a.proto` 的 `service A2AService`（11 个 RPC：SendMessage / SendStreamingMessage / GetTask / ListTasks / CancelTask / SubscribeToTask / CreateTaskPushNotificationConfig / GetTaskPushNotificationConfig / ListTaskPushNotificationConfigs / GetExtendedAgentCard / DeleteTaskPushNotificationConfig）
- 消费入口：`docs/specification.md`（3618 行 normative 规范）

### 核心模块 / 核心抽象
| 抽象 | 位置 | 说明 |
|---|---|---|
| `A2AService` | a2a.proto:19-140 | 11 个 RPC；HTTP binding 全部带 `/{tenant}/` 附加绑定（多租户） |
| `Task` / `TaskState` / `TaskStatus` | a2a.proto:167-219 | 核心工作单元；8 态状态机（SUBMITTED/WORKING/COMPLETED/FAILED/CANCELED/INPUT_REQUIRED/REJECTED/AUTH_REQUIRED） |
| `Message` / `Part` / `Role` | a2a.proto:221-277 | 通信单元（text/raw/url/data 四种 part；ROLE_USER/ROLE_AGENT） |
| `Artifact` | a2a.proto:279-293 | 任务输出容器（含 append/lastChunk 流式组装） |
| `AgentCard` + `AgentInterface` + `AgentCapabilities` + `AgentSkill` + `AgentCardSignature` | a2a.proto:334-467 | 自描述发现清单（身份/接口/能力/技能/JWS 签名） |
| `SecurityScheme`（APIKey/HTTPAuth/OAuth2/OIDC/mTLS 五类） | a2a.proto:495-646 | OpenAPI 3.2 风格判别联合；OAuth2 现代化（authorization_code+PKCE/client_credentials/device_code，implicit/password 已 deprecated） |
| `TaskPushNotificationConfig` + `AuthenticationInfo` | a2a.proto:324-332, 469-485 | 异步推送配置 |
| `StreamResponse` | a2a.proto:790-803 | SSE/推送统一负载（task/message/status_update/artifact_update） |

### 核心数据结构（28 message + 3 enum 中的关键）
- `Task{id, context_id, status, artifacts, history, metadata}` —— context_id 分组、artifacts 输出、history 多轮
- `Message{message_id, context_id, task_id, role, parts, metadata, extensions, reference_task_ids}`
- `SendMessageConfiguration{accepted_output_modes, task_push_notification_config, history_length, return_immediately}`
- `AgentInterface{url, protocol_binding, tenant, protocol_version}` —— tenant 不透明路由键
- `ListTasksRequest{context_id, status, page_size(1-100), page_token, history_length, status_timestamp_after, include_artifacts}` —— 分页过滤

### 核心状态
- **TaskState 状态机**（a2a.proto:187-208）：4 终态（COMPLETED/FAILED/CANCELED/REJECTED）+ 2 中断态（INPUT_REQUIRED/AUTH_REQUIRED）+ SUBMITTED/WORKING
- **任务不可变性**（docs/topics/life-of-a-task.md §Task Immutability）：终态任务不可重启；后续交互必须新建任务（同 contextId）——**协议级不变量**
- Message-vs-Task 双轨：stateless Message vs stateful Task；Hybrid Agent 用 Message 协商范围、Task 追踪执行；Task 一旦创建只能返回 Task

### 主要测试体系
- **无测试体系**（无单元/集成测试文件）。CI 11 个 workflow 均为工程治理类（super-linter、spelling、conventional-commits、links、release-please、docs、stale、issue-metrics）——**协议仓库用 lint/规范一致性替代行为测试**；规范一致性由 `specification/.api-linter.yaml` + buf 工具链保障

### 主要配置
- `specification/buf.yaml` + `buf.lock` + `buf.gen.yaml` —— protobuf 编译/生成（JSON 模式用 proto_to_json_schema.sh）
- `.github/workflows/` —— 11 个 CI
- `.github/super-linter.env` / `.github/linters/` —— lint 配置
- `mkdocs.yml` / `requirements-docs.txt` / `lychee.toml` —— 文档构建与链接检查
- `specification/.api-linter.yaml` —— API 规则

### 主要权限 / policy / governance 机制
- **GOVERNANCE.md（TSC 宪章）**：八席位技术指导委员会（Google/MS/Cisco/AWS/Salesforce/ServiceNow/SAP/IBM 各一席）；共识优先、投票多数；quorum 50%；六周不参会=inactive；GitHub 为重大决策 source of truth
- **扩展/绑定两级治理**（docs/topics/extension-and-binding-governance.md）：official（`ext-`/`cpb-` 前缀 + `https://a2a-protocol.org/…` URI 命名空间）vs experimental（`experimental-` 前缀，需 Maintainer 赞助）；生命周期 Proposal→Sponsorship→Experimental→Graduation（TSC 投票 quorum 50% + 多数）→Official→Promotion to Core；URI 是标识符不是 URL
- **安全政策**（SECURITY.md）：GitHub Security Advisories 私有报告
- **扩展安全铁律**（docs/topics/extensions.md §Security）：扩展不得绕过 agent 主安全控制；新增方法必须同等级 authn/authz；required:true 是硬依赖只能用于核心/安全功能

### 主要外部依赖
- protobuf / googleapis（google/api/annotations、client、field_behavior）——工具链依赖
- Linux Foundation（A2A 2026-08-20 捐赠/08-21 接纳；GOVERNANCE.md 引用 Series Agreement）
- 无运行时第三方依赖（规范库）
