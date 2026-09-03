# Project Layer（项目层 · 项目地图）

> **v3 三层架构的第一层。** 回答"Codex 到底是什么，怎么运行"——架构、模块、关键组件、配置、生命周期、依赖、入口。
> 这是"项目事实底座"，为 Engineering Knowledge 和 Generalized Knowledge 提供上下文。
> 复用 v1 repository-map（已独立核验），按 v3 三层架构重新组织。

---

## 一、项目定位

**一句话**：Codex CLI 是 OpenAI 出品的本地运行编程 Agent，通过 Rust 实现的 Agent 运行时，连接 LLM、工具执行、沙箱、MCP 协议，在终端中提供自主编程能力。

| 项 | 值 |
|----|-----|
| 项目 | openai/codex |
| 类型 | Agent 系统 / CLI 工具 / 运行时 |
| 语言 | Rust（112 crates，workspace 管理） |
| 构建 | Bazel + Cargo 双构建系统 |
| UI | ratatui（TUI） |
| 协议 | JSON-RPC（app-server）、MCP（Model Context Protocol） |
| 沙箱 | Linux（bwrap/seccomp）、macOS（Seatbelt）、Windows（自定义 sandbox service） |
| SDK | Python / TypeScript |

---

## 二、架构总览

```
                    ┌─────────────────────┐
                    │     CLI / TUI       │
                    │   (ratatui 界面)     │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │   App Server         │
                    │  (JSON-RPC API)      │
                    └──────────┬──────────┘
                               │
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
     ┌────────────┐  ┌────────────┐  ┌────────────┐
     │  Session    │  │   Agent     │  │  Context   │
     │  (turn/step)│  │ (multi-agent)│  │ (50+ frag) │
     └─────┬──────┘  └─────┬──────┘  └─────┬──────┘
           │                │                │
     ┌─────▼────────────────▼────────────────▼──────┐
     │              Core Runtime                       │
     │  ┌──────────┐ ┌──────────┐ ┌───────────────┐ │
     │  │ Guardian  │ │ExecPolicy│ │NetworkApproval│ │
     │  │(AI审批器) │ │(策略引擎) │ │ (网络代理回调) │ │
     │  └──────────┘ └──────────┘ └───────────────┘ │
     │  ┌──────────┐ ┌──────────┐ ┌───────────────┐ │
     │  │  Tools    │ │Rollout   │ │Permission     │ │
     │  │(registry/ │ │Budget    │ │ProfileState   │ │
     │  │ router)   │ │(token预算)│ │  (SSOT)       │ │
     │  └──────────┘ └──────────┘ └───────────────┘ │
     └───────────────────────┬────────────────────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
     ┌────────────┐  ┌────────────┐  ┌────────────┐
     │  Sandbox    │  │  Exec      │  │  MCP Server │
     │ (bwrap/     │  │ (执行环境   │  │ (工具扩展    │
     │  seatbelt)  │  │  抽象)      │  │  协议)       │
     └────────────┘  └────────────┘  └────────────┘
```

---

## 三、核心模块（A 级路径，高知识密度）

### 3.1 核心 Agent 运行时（`codex-rs/core/src/`，472 个文件）

| 模块 | 路径 | 职责 | 知识密度 |
|------|------|------|---------|
| Session | `session/` | Session 生命周期、turn 管理、step activation、token budget | ★★★★★ |
| Tools | `tools/` | 工具系统：registry/router/orchestrator/handlers（shell/apply_patch/mcp/multi_agents/plan） | ★★★★★ |
| Agent | `agent/` | 多 Agent：control/registry/role/residency/spawn/user_authorization | ★★★★☆ |
| Guardian | `guardian/` | Guardian 审批系统：approval_request/review/review_session/prompt/feedback | ★★★★☆ |
| Context | `context/` | 上下文片段系统：50+ 种 ContextualUserFragment，world_state 渲染 | ★★★★☆ |
| ExecPolicy | `exec_policy.rs` | 执行策略：策略持久化、热更新、危险前缀过滤、多层 config stack | ★★★★☆ |
| UnifiedExec | `unified_exec/` | 统一执行：process/process_manager/process_state/stdin_approval/shell_snapshot | ★★★★☆ |
| RolloutBudget | `rollout_budget.rs` | 跨 agent 树共享 token 预算 | ★★★★☆ |

### 3.2 执行与沙箱

| 模块 | 路径 | 职责 |
|------|------|------|
| Exec | `codex-rs/exec/` | 执行环境抽象 |
| ExecServer | `codex-rs/exec-server/` | 独立执行服务器（跨 OS 远程执行） |
| Sandboxing | `codex-rs/sandboxing/` | 沙箱抽象层 |
| LinuxSandbox | `codex-rs/linux-sandbox/` | Linux bwrap/seccomp 沙箱实现 |
| WindowsSandbox | `codex-rs/windows-sandbox-rs/` | Windows 沙箱 Rust 端 |

### 3.3 工具与协议

| 模块 | 路径 | 职责 |
|------|------|------|
| MCPServer | `codex-rs/mcp-server/` | MCP 服务器实现 |
| AppServer | `codex-rs/app-server/` | JSON-RPC API 服务器（IDE/桌面端集成） |
| NetworkProxy | `codex-rs/network-proxy/` | 网络代理（allowlist + 回调） |

### 3.4 模型与配置

| 模块 | 路径 | 职责 |
|------|------|------|
| ModelProvider | `codex-rs/model-provider/` | 模型提供抽象（OpenAI/兼容 API） |
| Config | `codex-rs/config/` | 配置系统（多层 config stack） |

### 3.5 治理文档

| 文档 | 路径 | 内容 |
|------|------|------|
| AGENTS.md | `AGENTS.md` | Agent 治理文档：编码规范、core crate 反膨胀、模型上下文规则（6 条硬约束）、测试规范、API 设计、变更大小限制（800 行） |

---

## 四、关键组件与生命周期

### 4.1 Session 生命周期

```
Session 创建
  ├── 加载 config_stack（builtin → user → project → requirements overlay）
  ├── 加载 exec_policy（default.rules）
  ├── 初始化 PermissionProfileState（SSOT）
  ├── 初始化 AgentControl（单例，clone 给 subagent）
  ├── 初始化 RolloutBudget（OnceLock 延迟初始化）
  └── 进入 turn 循环
       ├── Turn 开始
       │    ├── pending_reminder（RolloutBudget 阈值提醒）
       │    └── 构建上下文（50+ ContextualUserFragment）
       ├── Step 执行
       │    ├── 模型推理
       │    ├── 工具调用
       │    │    ├── exec_approval_requirement（三态决策）
       │    │    ├── Guardian 审查（如 OnRequest/Granular + AutoReview）
       │    │    ├── begin_network_approval（网络代理）
       │    │    ├── 沙箱执行
       │    │    └── append_amendment_and_update（策略固化）
       │    └── record_usage（RolloutBudget 计费）
       └── Turn 结束
            ├── record_guardian_denial/non_denial（熔断器更新）
            └── 持久化 thread_settings
Session 销毁
  ├── cancellation_token 触发（取消 ActiveNetworkApproval）
  ├── Weak\<Session\> upgrade 失败 → 回调保守降级
  └── PendingApprovalDecision drop → 默认 Deny
```

### 4.2 关键组件清单

| 组件 | 位置 | 职责 | 对应 EK |
|------|------|------|---------|
| ExecPolicyManager | exec_policy.rs:276 | 策略持久化与热更新 | EK-02, EK-09 |
| Guardian | guardian/ | AI 自动审批审查器 | EK-03 |
| NetworkApprovalService | tools/network_approval.rs | 网络审批闭环 | EK-04 |
| AgentControl | agent/control.rs | 多 agent 控制面单例 | EK-07, EK-11 |
| RolloutBudget | rollout_budget.rs | 跨 agent 树 token 预算 | EK-05 |
| PermissionProfileState | session/session.rs:96 | Session 级权限 SSOT | EK-06 |
| ContextualUserFragment | context/ | 上下文片段 trait（50+ 实现） | EK-08 |
| ExecApprovalRequirement | tools/sandboxing.rs:152 | 三态审批枚举 | EK-01 |
| ActiveNetworkApproval | tools/network_approval.rs:1153 | 活跃网络审批（cancellation_token） | EK-15 |
| PendingApprovalDecision | tools/network_approval.rs:300 | 待处理审批（drop 默认 Deny） | EK-26 |

---

## 五、关键配置

| 配置 | 位置 | 说明 | 对应 EK |
|------|------|------|---------|
| AskForApproval（四态） | sandboxing.rs:189 | Never/OnRequest/Granular/UnlessTrusted | EK-38 |
| BANNED_PREFIX_SUGGESTIONS（88 个） | exec_policy.rs:57-146 | 禁止自动固化的危险前缀 | EK-37 |
| Guardian 超时 90s / 重试 3 次 | guardian/mod.rs:62, review.rs:83 | 审批器超时与重试 | EK-39 |
| Guardian 5 层上下文预算 | guardian/mod.rs:71-77 | 20K/10K/5K/1K/40 | EK-22 |
| 熔断器阈值 Standard 3/10, Cyber 1/1 | guardian/mod.rs:64-68 | 连续/近期拒绝阈值 | EK-35 |
| 多层 config stack | exec_policy.rs:662 | builtin→user→project→requirements | EK-40 |
| 策略文件 default.rules | exec_policy.rs:54-56 | codex_home/rules/default.rules | EK-42 |
| AGENTS.md 6 条上下文硬约束 | AGENTS.md:91-100 | no rewrite / 10K cap / 1K P0 / trait | EK-24, EK-36 |
| effective_agent_max_threads | agent/control.rs:169 | 并发 agent 数限制 | EK-41 |

---

## 六、依赖与入口

### 6.1 入口

| 入口 | 路径 | 说明 |
|------|------|------|
| CLI | `codex-rs/cli/` | 命令行入口与命令解析 |
| TUI | `codex-rs/tui/` | ratatui 终端界面 |
| App Server | `codex-rs/app-server/` | JSON-RPC API（IDE/桌面端集成） |
| MCP Server | `codex-rs/mcp-server/` | Model Context Protocol 服务器 |

### 6.2 关键外部依赖

| 依赖 | 用途 |
|------|------|
| arc-swap | 无锁读 + 原子写（策略热更新） |
| tokio | async runtime（spawn_blocking、Semaphore） |
| ratatui | TUI 界面 |
| serde | 序列化/反序列化 |
| codex-execpolicy | 策略解析与匹配（外部 crate） |
| codex-network-proxy | 网络代理（allowlist + 回调） |

---

## 七、知识藏区标注（高知识密度预判）

1. **沙箱与执行边界**：`sandboxing/` + `linux-sandbox/` + `windows-sandbox-rs/` + `exec-server/` + `unified_exec/` —— Agent 系统最核心的安全边界
2. **Guardian 审批**：`core/src/guardian/` + `guardian-context/` —— 人类审批循环的实现
3. **多 Agent 控制**：`core/src/agent/control/` + `agent-roles/` + `agent-identity/` —— 多 Agent 协作与权限
4. **工具调度**：`core/src/tools/orchestrator.rs` + `router.rs` + `registry.rs` —— 工具调用的核心调度
5. **上下文工程**：`core/src/context/` + `context-fragments/` + `compact/` —— 50+ 上下文片段的组装与压缩
6. **执行策略**：`core/src/exec_policy.rs` + `config/permissions.rs` —— 什么能执行、什么需要审批
7. **MCP 协议**：`mcp-server/` + `codex-mcp/` —— 工具扩展协议
8. **AGENTS.md 治理规则**：core 反膨胀、上下文硬限制、变更 800 行限制 —— 工程决策的集中体现

---

> **Project Layer 与 Engineering Knowledge 的关系**：本层回答"Codex 是什么、有什么模块、怎么运行"；Engineering Knowledge 层（01-engineering-knowledge.md）回答"Codex 具体怎么解决工程问题"（核心机制/关键实现/决策/失败/配置/边界）。本层的组件清单和配置表为 Engineering Knowledge 提供索引和上下文。
