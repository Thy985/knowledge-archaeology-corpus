# Repository Snapshot Artifact — microsoft/agent-framework（ARCH-2026-09-29-001）

| 字段 | 值 |
|---|---|
| repository | https://github.com/microsoft/agent-framework.git |
| commit SHA | 95e711a6280d0ed7bafa5bafb8ff38eba4cffd3e |
| commit message | ".NET: Update version for 1.23.0 release (#8806)" |
| branch | main |
| snapshot timestamp | 2026-09-29T02:05:00+08:00（浅克隆） |
| files | 5412 |
| size | 136,525 KB（GitHub API languages 端点） |
| stars | 13841（2026-09-28 快照） |
| license | MIT |
| language | Python（dominant）· C# · TypeScript |

## 项目基础地图
- 语言：Python（core 1.19.0）/ C#（1.23.0）/ Go（README 存根）
- 入口：python/packages/core/agent_framework/（Python 包）；dotnet/src/Microsoft.Agents.*（C#）
- 核心模块：_agents.py / _harness/（loop, tool_approval, file_access, memory, mode, todo, background_agents）/ _workflows/ / security.py / orchestrations 包
- 核心数据结构：AgentSession（state dict）/ Message / Content / AgentResponse / ResponseStream / ToolApprovalState / MemoryTopicRecord / WorkflowRunResult
- 状态：session.state（provider-scoped）；审批状态/工具预算/后台任务/模式/todos
- 测试体系：core 96 测试文件 + orchestrations 8；harness 专项 10 文件
- 配置：pyproject（core 1.19.0）；_settings.py（env/文件源 + SecretString）；_feature_stage.py（实验特性标记）
- 权限与治理：ToolApprovalMiddleware（审批规则/预算/二次审批）+ FIDES（标签传播/策略执行/变量隔离/quarantine）+ FileAccess（路径规范化）+ MCP 自动标签
- 外部依赖：msgspec/opentelemetry/pydantic（core）；按供应商集成包；hyperlight（CodeAct 沙箱，linux x86_64/win AMD64）；azure-cosmos-agent-memory（Python 3.11+）
- 重要设计证据：docs/decisions/（41 ADR），0024 FIDES / 0038 CodeAct / 0037 Skills / 0019 Compaction
