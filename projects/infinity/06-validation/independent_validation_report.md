# Independent Validation Report — AWS Infinity（盲重建）

> **盲重建方法**：Auditor 不阅读考古 Package（00-06）原文，独立重读仓库源码/文档/测试，建立 Independent Findings 后再与考古产物对比。
> **目标攻击点**：单案例→Pattern、Pattern→L4、项目经验→通用 Principle、ADR→实现事实、Flow Edge 真实性、bypass/override/exception/alternate/direct call/admin path/fallback/legacy path、Epistemic 混淆、计数类断言。

## 1. 独立抽查记录（Blind Reconstruction，不参考考古产物）

| # | 独立重读对象 | 独立发现 |
|---|---|---|
| B-1 | `event_processor.rs:63-99` | `PrepareResult` 五变体（Ready/Handled/OAuthRequired/UserChoiceRequired/CompactionApplied）；`CompletionAction::{Done,ExecuteToolCall}` —— 与考古 EK-01 一致 |
| B-2 | `system/thread.rs:248-297` `filter_deferrable` | **新细节**：deferral 精确顺序 = batch 先于 drained deferred 项；`settles_pending` 判定 = 批内含 pending call 的真实结果或用户文本（中断）才 flush deferred；被动工具判定经 `resolved_config` |
| B-3 | `tools/thread.rs:65-140` | spawn_thread 同步执行、child_of 校验、fresh_context 分支 —— 与考古 EK-10/11 一致 |
| B-4 | `tools/thread.rs:345-420` | **新细节（考古遗漏）**：`close_thread` 禁止关闭 root 线程（"cannot close the root thread. Only child threads can be closed."）；thread_id 必须匹配当前线程 |
| B-5 | `tools/thread.rs:249-300` | **新细节（考古遗漏）**：`report_to_parent` 检测 `is_subscription_event_thread` → 特殊前缀 "Report from temporary child thread created to process a subscription event" |
| B-6 | `system/thread.rs:52-63` | `is_user_text_input` / `is_deferrable_synthetic_event` —— 与考古 EK-20 一致 |
| B-7 | `event_processor.rs:627-673` `sync()` | **新细节（精确化）**：sync 只把 `User` 与 `SubscriptionEvent` 类型的 message ID 写入 processed-ID 集（filter 显式 `matches!`）——考古 EK-06 语义一致但未精确到类型级 |
| B-8 | `tools/mod.rs:60-120` | **新细节（考古遗漏）**：`Tool` trait 实际 6 个方法：name/description/parameters/execute/supports_sync/is_passive/display_script（Rhai 可选）；`safe_spawn_point: Option<usize>` 只同步工具期间 set |
| B-9 | `provider-protocol/src/remote.rs:1-50` | **新细节（考古遗漏）**：进程外 provider 走 **Unix domain socket + JSON-lines**（`serve_provider`/`UnixListener`/`LinesCodec`），非 HTTP |
| B-10 | `tools/rap_tool.rs:72-135` | RAP 调用错误路径：非 2xx → `send_tool_error`（"No result will arrive for this call"）；thread_ancestors 只在 >1 层栈时传 —— 与考古 EK-04 一致 |
| B-11 | `mcp-bridge/src/lib.rs:1-40` | **关键差异（考古 EK-29 需修正）**：bridge 实际是 `McpToolSet`（lazily connected stdio/Streamable HTTP MCP server → **暴露为本地 Infinity tools**）+ `McpClient`（共享传输层）——不是"MCP 服务器通过兼容层直接跑 RAP"，而是**把 MCP 服务器适配为本地工具** |
| B-12 | `tools/sleep.rs:35,83,158` | 三个 sleep 工具全部 `is_passive=true` —— 与考古 EK-20/06 一致 |
| B-13 | `agent_scale.rs:171-191` | RSS 采样 = `/proc/self/status` VmRSS + `libc::malloc_trim(0)` —— 与考古 EK-36 方法学一致 |
| B-14 | `architecture.md:8-121` | slice 不变量/挂起-恢复/FIFO/幂等/compaction 原文与考古 KO-01/02/04 一致；"roughly three quarters" 与考古 EK-17 ">75%" 一致 |
| B-15 | `step-mode.md:91-101` | batch size>1 时 SQS FIFO 可交错多 message group；step 按线程分区并发跑 slice —— 与考古 Flow 一致 |

## 2. 判定统计

| 判定 | 数量 | 明细 |
|---|---|---|
| CONFIRMED | 10 | B-1/B-3/B-6/B-7/B-10/B-12/B-13/B-14/B-15 + 计数类 |
| PARTIALLY_CONFIRMED | 1 | B-11（MCP bridge 机制描述偏差） |
| DOWNGRADED | 0 | — |
| OVER_GENERALIZED | 0 | — |
| MISSING | 4 | B-4（close_thread root 禁令）/ B-5（订阅线程报告前缀）/ B-8（Tool trait 6 方法+Rhail display）/ B-9（provider Unix socket 传输） |
| CONTRADICTED | 0 | — |
| NEEDS_HUMAN_REVIEW | 2 | C-02（slice 通用性）/ C-05（中断注入毒化） |

## 3. 考古产物 vs 盲重建对比

### 3.1 需修正（1 处，不推翻核心）

**EK-29（MCP bridge）描述偏差（PARTIALLY_CONFIRMED → 修正为准确描述）**
- 考古写："现有 MCP 服务器通过兼容层直接运行（不修改）"，暗示协议级兼容。
- 盲重建实际：`McpToolSet` 把 MCP 服务器（stdio/Streamable HTTP）**暴露为本地 Infinity tools**，`McpClient` 是共享传输层。即：**MCP 服务器以工具身份接入 agent-core，不是以 RAP 协议服务器身份接入**。
- 影响：不影响 KO 结构；修正 EK-29 表述（03 的 KO 无引用 MCP bridge 的聚合，安全）。

### 3.2 需补充（4 处，考古 Package 遗漏的新细节）

1. **B-4 close_thread 边界**：禁止关闭 root；thread_id 必须匹配当前线程 → 补充 EK（属 EK-10 的约束边）。
2. **B-5 订阅线程报告前缀**：订阅事件临时线程的 report 带特殊标注 → 补充 EK-15 的机制细节。
3. **B-8 Tool trait 完整面**：`supports_sync`/`is_passive`/`display_script`（Rhai）+ `safe_spawn_point` → 补充 EK-03 与 EK-32 的契约细节。
4. **B-9 provider Unix socket**：进程外 provider 用 Unix socket JSON-lines（非 HTTP）→ 补充 EK-22 的平台抽象细节（provider 传输与 RAP 工具传输不同）。

### 3.3 精确化（1 处）

**EK-06 幂等去重的类型级精确化**：sync() 只把 `InfinityMessage::User` 与 `SubscriptionEvent` 的 ID 入 processed 集（代码 filter 显式匹配，非全部"非天然幂等"）。

## 4. 攻击结果（按 skill 要求 3 成功 + 3 错误 + 遗漏 + Benchmark case）

### 4.1 攻击成功的断言（3）

1. **"MCP bridge = 协议兼容层"攻击成功**：盲重建证明 bridge 是"适配为本地工具"而非"协议兼容"，考古描述不准确（见 3.1）。
2. **"sync 幂等 ID 覆盖全部非天然幂等输入"攻击成功**：代码精确化显示只覆盖 User + SubscriptionEvent（见 3.3）。
3. **"Tool trait 基础面=name/desc/params/execute"攻击成功**：实际 6 方法 + Rhai display_script + safe_spawn_point（见 B-8）。

### 4.2 攻击失败（考古正确的断言，3）

1. **"slice 三阶段"**：盲重建从 event_processor.rs 独立确认三函数存在与签名 —— 攻击失败，考古正确。
2. **"spawn_thread 同步 + child_of 校验"**：独立确认 —— 攻击失败。
3. **"compaction >75% 触发"**：architecture.md "roughly three quarters" + driver.rs ">75%" 双源一致 —— 攻击失败。

### 4.3 遗漏（4）

见 3.2（close_thread root 禁令 / 订阅线程报告前缀 / Tool trait 全貌 / provider Unix socket）。

### 4.4 Benchmark case（本环境无 Docker，验证方式说明）

- **静态 benchmark 复现**：计数类断言（168 文件/57,187 行/21 crates/28 测试/150 snapshots）已用独立脚本重数 ✓（与考古一致）。
- **运行时 benchmark**（50k agents）无法复现（无容器/无 Rust 工具链运行环境）→ 维持 S4（仓库 CHANGELOG 自述），不升 S5。

## 5. Reconciliation 建议（供阶段⑥合并进 Corpus）

1. **修正 EK-29**：MCP bridge = `McpToolSet` 将 MCP 服务器（stdio/Streamable HTTP）暴露为本地 Infinity tools；`McpClient` 共享传输层；非"协议兼容直跑"。
2. **补充 EK-46**（close_thread 边界）：禁止关闭 root；thread_id 必须匹配当前线程；错误返回而非静默。
3. **补充 EK-47**（订阅线程报告标注）：`report_to_parent` 检测 `is_subscription_event_thread` → 前缀 "Report from temporary child thread created to process a subscription event"。
4. **补充 EK-48**（Tool trait 完整契约）：`supports_sync` / `is_passive` / `display_script`（Rhai）/ `safe_spawn_point`；被动工具自由中断 vs 非被动 deferral。
5. **补充 EK-49**（provider 进程外传输）：Unix socket + JSON-lines（serve_provider / UnixListener / LinesCodec）；与 RAP HTTP 工具传输对比。
6. **精确化 EK-06**：processed-ID 集仅覆盖 `User` + `SubscriptionEvent` 消息类型（代码级 filter）。
7. **KO 影响评估**：全部 9 KO 不受上述修正影响（无 KO 聚合引用 EK-29 或新增细节的簇）；核心命题（挂起-恢复/幂等/FIFO/synthetic/平台抽象）全部 CONFIRMED。

## 6. 最终结论

- **核心考古结论全部成立**：挂起-恢复执行模型、slice 三阶段、幂等+FIFO 无锁并发、synthetic tool calls、平台 4-trait 可移植、context 治理、错误分类恢复 —— 盲重建独立确认。
- **1 处描述修正**（MCP bridge 机制）+ **4 处细节补充** + **1 处精确化**，均不推翻核心，已在 Reconciliation 列出。
- **证据上限维持 S4**；无 DOWNGRADED / CONTRADICTED。
- **NEEDS_HUMAN_REVIEW 2 项**：C-02（slice 语义通用性）、C-05（中断注入模型毒化）。
