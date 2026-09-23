# 01 — Project Layer（项目地图）

> 本层 = L0 工程事实底座。所有事实可追溯到 `repo/` 实际内容（commit `43e0c874`）。

## 1.1 项目定位
- 跨 AI 生态（LangGraph/CrewAI/ADK/Genkit 等框架）的 **Agent-to-Agent 互操作协议**，2026-08-20 由 Google 捐赠 Linux Foundation（docs/llms.txt）
- 三部分构成（GOVERNANCE.md §Mission）：① Agent2Agent Protocol ② 实现用 SDK（独立仓库）③ 文档/其他产物

## 1.2 架构
```
抽象层（规范，唯一资产）        specification/a2a.proto（812 行）
  └─ A2AService（11 RPC，含 /{tenant}/ 多租户 HTTP 绑定）
  └─ 28 message + 3 enum（Task/Message/Part/Artifact/AgentCard/SecurityScheme/…）
规范性文档                    docs/specification.md（3618 行，RFC 2119）
  └─ 编号锚点（§8 Agent Discovery / §3111 Get Extended Agent Card / §431 TaskPushNotificationConfig…）
传输绑定                       JSON-RPC / gRPC / HTTP+JSON（标准三绑定，可自定义）
治理层                        GOVERNANCE.md（TSC 八席位）+ 扩展/绑定两级治理
SDK（不在本仓库）             a2a-sdk（PyPI/README 徽章指向）
```
**关键架构事实**：v1.0.0 大重构 = "separate application protocol definition from mapping to transports"（CHANGELOG #b078419）——应用协议定义与传输映射分离。

## 1.3 核心模块 / 抽象
| 抽象 | 位置 | 职责 |
|---|---|---|
| A2AService | a2a.proto:19-140 | 11 RPC：Send/Stream/Get/List/Cancel/Subscribe/PushConfig×4/ExtendedCard |
| Task/TaskState/TaskStatus | :167-219 | 有状态工作单元 + 8 态状态机 |
| Message/Part/Role | :221-277 | 无状态通信单元；part 四类型（text/raw/url/data） |
| Artifact | :279-293 | 任务输出；流式用 append/last_chunk 分块组装 |
| AgentCard 族 | :334-467 | 身份/接口/能力/技能/签名（JWS）自描述清单 |
| SecurityScheme 族 | :495-646 | 五类认证方案（APIKey/HTTP/OAuth2/OIDC/mTLS）判别联合 |
| TaskPushNotificationConfig | :469-485 | 异步推送配置（url/token/authentication） |
| StreamResponse | :790-803 | SSE/推送统一负载（task/message/status/artifact_update） |

## 1.4 生命周期（协议视角）
- **任务生命周期**：Message（协商）→ Task 创建（SUBMITTED→WORKING→…）→ 中断态（INPUT_REQUIRED/AUTH_REQUIRED，需客户端输入）→ 终态（COMPLETED/FAILED/CANCELED/REJECTED，不可逆）
- **协议版本生命周期**：v0.2.1(2025-05-27) → v0.2.2(2025-06-09) → v0.2.3/0.2.4/0.2.5/0.2.6 → v0.3.0(2025-07-30) → **v1.0.0(2026-03-12，破坏性重构)** → v1.0.1(2026-05-26)
- **扩展生命周期**：Proposal → Maintainer Sponsorship → Experimental（`experimental-ext-`）→ Graduation（TSC 投票）→ Official（`ext-`）→ Promotion to Core

## 1.5 核心数据结构（字段级锚点）
- `Task{id REQ, context_id, status REQ, artifacts[], history[], metadata}`（a2a.proto:167-184）
- `TaskStatus{state REQ, message, timestamp}`（:210-219）
- `Message{message_id REQ, context_id, task_id, role REQ, parts REQ, metadata, extensions[], reference_task_ids[]}`（:260-277）
- `AgentInterface{url REQ, protocol_binding REQ, tenant, protocol_version REQ}`（:336-355）
- `SendMessageConfiguration{accepted_output_modes[], task_push_notification_config, history_length optional, return_immediately}`（:143-161）
- `ListTasksRequest{tenant, context_id, status, page_size(1~100,默认50), page_token, history_length, status_timestamp_after, include_artifacts}`（:676-701）

## 1.6 状态机（精确）
- `TaskState`（a2a.proto:187-208）：UNSPECIFIED(0)/SUBMITTED(1)/WORKING(2)/COMPLETED(3)/FAILED(4)/CANCELED(5)/INPUT_REQUIRED(6)/REJECTED(7)/AUTH_REQUIRED(8)
- **终态** = COMPLETED/FAILED/CANCELED/REJECTED；**中断态** = INPUT_REQUIRED/AUTH_REQUIRED；SUBMITTED/WORKING 非终态
- 规范注释：SubscribeToTask 对已终态任务返回 UnsupportedOperationError（a2a.proto:75）

## 1.7 配置
- `specification/buf.yaml/buf.lock/buf.gen.yaml`：protobuf 工具链
- `.github/workflows/` 11 个 CI：docs/conventional-commits/check-linked-issues/dispatch-a2a-update/issue-metrics/links/linter/release-please/sort-spelling-allowlist/spelling/stale
- `mkdocs.yml`/`requirements-docs.txt`/`lychee.toml`：文档构建+链接检查
- `specification/.api-linter.yaml`：API 规则

## 1.8 权限与治理机制
- TSC 八席位：Google/Microsoft/Cisco/AWS/Salesforce/ServiceNow/SAP/IBM 各 1 名投票成员（GOVERNANCE.md:3-14）
- 投票：共识优先；quorum ≥50%；缺席会议 ≥6 周（LFX 考勤）→ inactive，不计入 quorum
- 扩展/绑定治理：两级（official/experimental）+ 全生命周期（见 1.4）；官方 URI 命名空间 `https://a2a-protocol.org/extensions/` 与 `…/bindings/`（标识符，HTTP 访问不预期）
- 安全报告：GitHub Security Advisories（SECURITY.md）

## 1.9 外部依赖
- protobuf + googleapis annotations（工具链依赖）
- Linux Foundation（Series Agreement / LFX 平台）
- 无运行时第三方依赖（规范库；SDK 独立仓库）
