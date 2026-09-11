# 02 — Engineering Knowledge（EK Graph，v0.1.5 REFRESH）

> 规则：每条 EK 声明 `links`（六类边：mechanism/subsystem/causal/dependency/constraint/contrast）。基线 EK-01..32 的完整正文见 09-03 考古包；本节先给出**存续性核对**，再新增 **EK-R01..R13**（以 `.agents/notes` ADR 体系为主证据源——09-03 零引用的那块）。

## 2.0 基线 EK 存续性核对（v0.1.5）

| EK | 主张 | v0.1.5 状态 | 依据 |
|---|---|---|---|
| EK-01..06 | turn/step、append-only 日志、JSONL、surface 投影、增量折叠、双版本纪律 | **成立**（V3 envelope 强化 EK-02/05） | 2026-09-06-v3-canonical-session-envelopes |
| EK-07 | 受控工具执行管线 | **成立**（PTC collapse 位置验证"先于策略管线"） | 2026-08-07-ptc-executor-collapse |
| EK-08..14 | 超时/guard/approval/sandbox/credential/presets | 成立（本轮未发现反证） | 07-19-cooperative-tool-cancellation 等 |
| EK-15 | Landlock ABI 教训 | 成立（后续 windows-acl-restricted-token 扩展） | 2026-08-08-windows-acl-restricted-token-sandbox |
| EK-16..26 | adapter 注册表/错误族/prompt/compaction/max-tokens/subagent/外部 agent/seam/插件内核/invariants/启动强制 | 成立 | notes 大量佐证（twin-llm-adapters 等） |
| EK-27..32 [R] | approval 委派/PTC collapse/plan-mode/settings seam/repeat-tool-reminder/legacy 硬拒绝 | **成立且本轮深化**（EK-R03 补 collapse 因果链） | 本轮 notes |

## 2.1 新增 EK（本轮，主证据源 = .agents/notes）

### EK-R01 — Agent Notes = 显式决策证据源（ADR 体系）
**Fact/Observation**：`.agents/notes/implemented/{architecture,feature,bug-fix,process,simplification,testing}/` 958 条日期化笔记（09-03 时 862），每条 Problem→Decision→Alternatives considered 结构，双语配对（.zh.md + .i18n.yaml），archived/ 留存废弃决策。
**Why**：决策过程与实现同步入库；"frozen archive"（07-26 process note）保证笔记不可变。
**links**：`mechanism` EK-R01↔EK-24（决策记录=可插拔系统治理底座）；`subsystem` EK-R01↔EK-R02↔EK-R05（.agents/notes 族）；`constraint` EK-R01→EK-R06（改名纪律约束）。

### EK-R02 — PTC 演化链：设计→并行→collapse→改名（完整因果）
**链**：06-15 设计（模型写 TS 对工具注册表，worker_threads 沙箱，Cloudflare Code Mode 启发）→ 07-10 parallel → 07-20 typed returns → 07-26 live parallel dispatch → 08-07 executor collapse 修复 → 08-25 rename code-mode→ptc（无兼容别名，user-facing "PTC mode"/"PTC 模式"）。
**Why**：LLM 更擅长写代码而非发 tool-call；模式呈现属于注册表（避免 waterfall 变换依赖 listener 顺序）；pre-release 改名必须全量原子。
**links**：`causal` EK-R02→EK-R03（collapse 是演化终点）；`contrast` EK-R02↔EK-01（工具调用 vs 代码编写两种呈现）；`subsystem` EK-R02↔EK-16。

### EK-R03 — PTC executor collapse：schema 省略≠强制，拒绝必须经 executor（深化 EK-28）
**Fact**：`wireSchemas()` 只通告 `run_code`，但旧 executor 经 `get()` 放行全部工具（bypass 路径：模型发 `write/read/bash` 直接执行）。修复：`resolveExecution(name,scope,nested)` + `collapses(name,nested)`；`nested=false`（模型直接调用）仅允许 `run_code`，其余折叠为 `UNKNOWN_TOOL`（错误消息回指 run_code）；`nested=true`（仅 run_code SDK 绑定设置 parent token）可见全部工具。
**关键边界**：折叠在 `createExecution`（`prepare` 第一阶段）——**先于** `tools/pre-execute` 监听、approval `ask`、guards；人类永不被打扰审批一个确定性拒绝；不可 JSON 序列化参数报 TypeError（invalid-args 契约）而非 UNKNOWN_TOOL。
**Why**：包契约明示"schema omission is not enforcement when a direct caller can bypass it; denial must be tested through the executor"。
**links**：`dependency` EK-R03 依赖 EK-10（approval fail-closed 同族）；`causal` EK-R03→EK-07；`constraint` EK-R03→EK-31（repeat-tool-reminder 只对可见工具生效）。

### EK-R04 — workspace-files 服务：文件读取与导航的职责分离
**Fact**：`packages/api/workspace-files` 拥有 Host `ctx.workspaceFiles` + `workspaceFiles` Remote namespace + Client `file` provider；七方法（read/readBytes/readAll/readRelated/stat/list/changes）首参 `WorkspaceFileScope`，Gateway 从 wire Session 身份解析（live header 或 `SessionPersistence.stat`），**不激活 Agent、不读事件体、不回退父 Session**。内容按 page/byte-window/complete-file cap 限定；新 dsh-fs seam `FileSystem.readByteRange` 由所有 provider 实现。结果按绝对路径命名。Session Controller 不再承载 workspace-file 代码（模块归属纪律）。
**links**：`subsystem` EK-R04↔EK-R05↔EK-R06（workspace-files 族）；`mechanism` EK-R04↔EK-23（Capability Seam 三件套实例）；`dependency` EK-R04 依赖 EK-16（adapter 注册表装载）。

### EK-R05 — workspace-file 读权限演进：containment → 继承 fs 授权（设计超驰）
**Fact**：09-05 初版决策"文件方法 workspace-contained"；09-09 显式 **supersede**：`read/readBytes/readAll/readRelated/stat` 继承 Session fs 后端读授权（workspace 根=相对路径基准，非读边界；绝对路径与 `..` 越界可读，当后端允许）；`list/changes` 保持 workspace-scoped（导航/观察≠具名读取）；`readRelated` 从基文件目录解析相对路径。
**Why（Alternatives considered 记录）**：严格 containment 给预览一个比 Session fs 更窄的策略、阻断显式可读文件、破坏 HTML 外链渲染；CSP 禁 iframe 网络则拒绝有意保留的行为。Document Preview 打包静态声明 JS/CSS 进 HTML Blob iframe `sandbox="allow-scripts"`（opaque origin 防父访问，网络保留）——**intentional security trade-off**。
**links**：`contrast` EK-R05↔EK-R04（两版设计的对照）；`causal` EK-R05→EK-R08（Document Preview 依赖此授权）；`constraint` EK-R05 约束 EK-12（sandbox 三态：读授权来自 fs 后端而非 workspace 边界）。

### EK-R06 — 重命名纪律：pre-release 原子改名，无兼容别名
**Fact**：code-mode→ptc 一次性改 config 值、preset 目录、源码/测试文件名、root demo；**没有**兼容别名。命名理由："transport is not a sibling of plan-mode, so the identifier does not carry `-mode`"。
**links**：`mechanism` EK-R06↔EK-06（双版本纪律：格式 pinned）；`contrast` EK-R06↔EK-32（legacy 硬拒绝 vs 迁移兼容）。

### EK-R07 — Session 格式 V3：canonical envelope（surfaceOp 必填）
**Fact**：V3 要求 system/user/assistant message 与 tool/result 携带 `surfaceOp` = `'append'` 或 `{op:'replace',startSeq,endSeq}`（端点=当前 surface 序、含端、inclusive、无别名）；known log-only events 仅允许 type/seq/time/data/ignorable；assistant message 嵌入精确 provider stream 且**单独禁止** `sourceEventSeqs`；system/user/tool 可引用非空唯一较早 source 序列。接受规则：成员资格、有序端点、完整引用覆盖、content-only 单节点 tool-result 替换。**拒绝** `request/header.header.system` 与空 `tools:[]`/`adapterDefaults:{}`。`tool/result` 带 `data.error` 要求 `message.content[0].isError === true`；**不**从矛盾元数据推断错误结局。
**Why**：跨 in-memory/durable/browser-wire 多读取者，缺省 placement 或冲突元数据会导致静默漏消息与重建分歧。
**links**：`mechanism` EK-R07↔EK-02（append-only 唯一事实源）；`constraint` EK-R07 约束 EK-05（投影折叠基于 canonical 序）；`causal` EK-R07→EK-32（legacy 格式硬拒绝）。

### EK-R08 — Document Preview 安全权衡：opaque-origin iframe + 静态资产打包
**Fact**：预览将 bounded、静态声明的本地 JS/CSS 打包进 HTML Blob iframe（`sandbox="allow-scripts"`）；opaque origin 阻断父应用访问，浏览器保留正常网络访问。归属：Workspace Files 拥有分页/文件检查/list/observation；Document Preview 拥有"打包哪些相关文件 + iframe sandbox"。
**links**：`dependency` EK-R08 依赖 EK-R05；`constraint` EK-R08→EK-10（fail-closed：预览不是父文档权限提升）。

### EK-R09 — Python SDK：subprocess JSON-RPC over stdio + 显式 home 纪律
**Fact**：`deepseek-harness-sdk` 无独立入口，启动捆绑 `dsh` CLI（`--profile sdk`）；每次启动需显式 `dsh_home`/`DSH_HOME`，**故意不发现 `~/.dsh`**；同版本 `deepseek-harness-runtime-bin` wheel；uv.lock 锁定。**测试实测：111 passed / 7 skipped（3.35s）**。
**Why**：SDK 是"最小变体 benchmark 载体"（BENCHMARK.md）；显式 home 防隐式状态污染。
**links**：`mechanism` EK-R09↔EK-22（外部集成 seam：jsonrpc-agent）；`contrast` EK-R09↔EK-24（Python 子进程 vs Cordis 插件内嵌）；`dependency` EK-R09 依赖 EK-16。

### EK-R10 — SAFETY.md：experimental 责任声明（治理护栏）
**Fact**：显式声明——developer-preview、未经安全审计、不可视为 secure/production-ready；沙箱/审批/权限控制**不保证隔离**；责任使用（最小权限、一次性 VM/容器、备份、审查插件/配置/命令）。BENCHMARK.md：独立 workspace+session 跑 benchmark。
**links**：`constraint` EK-R10 约束 EK-12（sandbox 三态：官方明确其局限）；`mechanism` EK-R10↔EK-25（package-owned invariants：治理声明本身是包级不变量）。

### EK-R11 — 反馈体系：canonical-feedback-log + OTEL 双轨
**Fact**：09-05 canonical-feedback-log（规范化反馈日志）+ nonofficial-feedback-otel（非官方遥测）；08-25 feedback-gated-telemetry-default（反馈门控遥测默认关）；09-10 symmetric-message-feedback-submission + feedback-dialog-and-categories。
**links**：`subsystem` EK-R11↔EK-02（反馈作为事件族）；`contrast` EK-R11↔EK-01（用户反馈 vs 模型回合两条输入轨）。

### EK-R12 — 子代理治理细化：policy 继承 + 非交互权限 + model-selected routes
**Fact**：07-25 subagent-policy-inheritance（策略继承）、08-10 subagent-approval-pinned-never、08-15 product-subagent-noninteractive-permissions、08-18 model-selected-subagent-routes、08-24 user-authorized-subagent-model-routes、09-01 parent-owned-subagent-catalog。委派权限从"父 preset 继承"到"模型可选路由但用户授权"演进。
**links**：`causal` EK-R12→EK-21（Subagent 委托 seam 深化）；`mechanism` EK-R12↔EK-27（approval 委派继承）。

### EK-R13 — 会话持久化简化：jsonl-only + handle-based（09-03 后收敛）
**Fact**：08-30 jsonl-only-session-persistence（JSONL 唯一持久化形态）、08-27 handle-based-session-persistence、08-10 session-log-version-mechanism、08-31 released-session-format-migrations、09-02 projcache-cross-version-read-compat、09-05 read-only-session-migration-preparation。
**links**：`constraint` EK-R13 约束 EK-03（JSONL 只追加）；`mechanism` EK-R13↔EK-06（版本纪律：migration 显式化）。

## 2.2 EK Graph 质量自检

- EK 总数：32（基线）+ 13（新增）= 45，**全部有出边**（平均出边 ≥1）。
- 孤立 EK 比例：0.0。
- 聚合规则覆盖率：100%（见 03 层）。
