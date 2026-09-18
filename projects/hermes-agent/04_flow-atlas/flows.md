# Flow Atlas — 七类流（hermes-agent @ fba4cb1）

> 关键 Edge 必须可回溯到实际 symbol/file/condition/state transition。全部从快照仓库代码导出。

## 1. Control Flow（控制流）
```
cli.py → hermes_bootstrap（首 import，Windows UTF-8）
   → hermes_cli.* mixins（CLIAgentSetupMixin → CLIAgent 装配）
   → agent/agent_init.py（agent 初始化：加载 MEMORY.md/USER.md 冻结快照）
   → agent/prompt_builder.py（构建系统提示词：SOUL.md + 记忆块 + 工具 schema）
   → 主循环（CLILoopsMixin）：用户输入 → model_tools 分发 → tool handler → 响应
   → context_compressor（超预算时压缩，唯一 prompt 变更例外）
   → 会话结束 → memory 写盘（不影响当前 prompt）
```
- Edge 证据：cli.py:1-14（bootstrap import）；agent_init.py:1244（记忆加载）；AGENTS.md（cache 不变量）

## 2. State Flow（状态流）
```
state.db（SQLite WAL）
  ├─ session metadata / message history / model config / FTS5 索引
  ├─ parent_session_id 链（compression 分裂会话）
  ├─ source-tagged（cli/telegram/...）
  └─ 守护：hermes_state_guard（生产/测试隔离）→ hermes_state_wal（WAL/DELETE 回退）→ hermes_state_readpool（只读池）
```
- Edge 证据：hermes_state.py docstring（WAL/FTS5/parent_session_id）；hermes_state_wal.py（_WAL_INCOMPAT_MARKERS → fallback DELETE）；hermes_state_guard.py（_STATE_DB_GUARD_BYPASS_ENV）

## 3. Data Flow（数据流）
```
用户输入 → agent 上下文（冻结快照：MEMORY.md/USER.md § 卡片）
   → model API（providers/*，35 直接依赖 + 45 optional groups）
   → tool 分发（registry.register() 自动发现 → handle_function_call() → JSON string）
   → 工具写盘（checkpoint_manager 先快照 shadow git → 文件变更）
   → 轨迹记录（trajectory_compressor：保护 head+尾，不拆 tool_call/response 对）
   → durable transcript（SQLite）→ rewind（回滚权威）
```
- Edge 证据：tools/AGENTS.md（registry 发现/JSON 返回）；tools/checkpoint_manager.py（GIT_DIR/WORK_TREE/INDEX_FILE）；trajectory_compressor.py docstring；hermes_state_rewind.py（durable 权威）

## 4. Evidence Flow（证据流）
```
工具结果 → tool handler（JSON string）
   → hook_output_spill（spill_if_oversized：超限外溢）
   → 上下文压缩提示（"persistent memory ALWAYS authoritative"）
   → 审计轨迹（file_safety：可见 audit trail；approval 审批记录）
   → checkpoint ledger（agent-write ledger，shadow git）
```
- Edge 证据：tools/hook_output_spill.py（存在性）；agent/context_compressor.py:4603；agent/file_safety.py docstring；tools/checkpoint_manager.py（ledgers/<hash16>.json）

## 5. Authority Flow（权威流）
```
技能安装：skills_guard.scan_skill → should_allow_install（信任分级）
   ├─ builtin → allow（never scanned）
   ├─ trusted（openai/anthropics/hf/NVIDIA）→ caution allow / dangerous block
   ├─ community → caution block（--force 可覆盖）
   └─ agent-created → dangerous="ask"（报错重试）
文件写：file_tools_write_guards（SOUL.md/config.yaml 保护指令审批门）
工具执行：approval* 族（approval_smart/approval_floors/approval_gateway_wait）
执行域：terminal 默认直接宿主 → SECURITY.md 唯一承重边界=OS 隔离（容器/云沙箱/远程后端）
```
- Edge 证据：tools/skills_guard.py INSTALL_POLICY；agent/prompt_builder.py（file_tools_write_guards 引述）；SECURITY.md §2.2

## 6. Memory Flow（记忆流）
```
session 起始 → 冻结快照（MEMORY.md/USER.md → 系统提示词，不可变）
session 中 → memory 工具（add/replace/remove）→ 写盘（memory_tool_store，fcntl/msvcrt 锁）
   → MemoryManager fan-out → 外部 provider（唯一，prefetch/sync_turn/on_* hooks）
   → pre_compress checkpoint（v2 fail-closed / v1 best-effort）
session 间 → learning_graph（§ 拆卡片）→ learning_mutations（memory:<source>:<index>）
   → Curator（空闲：pin/archive/consolidate/patch，.curator_state）
```
- Edge 证据：tools/memory_tool.py（FROZEN snapshot）；agent/memory_provider.py（PRE_COMPRESS_CHECKPOINT_API_VERSION=2）；agent/learning_graph.py:130-134；agent/curator.py（.curator_state）

## 7. Policy Flow（策略流）
```
AGENTS.md 不变量（cache 神圣 / narrow waist）→ 开发治理
SECURITY.md §2.2（OS 隔离唯一边界）→ 安全策略 → 报告范围界定（§3）
skills_guard 信任策略（builtin/trusted/community/agent-created）→ 安装策略 → --force 逃生舱
Curator 策略（interval/stale/archive 天数）→ .curator_state 持久化 → 空闲触发
CONTRIBUTING.md 技能作者标准 → skill_linter 咨询性建议
```
- Edge 证据：AGENTS.md（两不变量）；SECURITY.md（§2.2/§3）；tools/skills_guard.py（INSTALL_POLICY）；agent/curator.py（DEFAULT_STALE_AFTER_DAYS=14/ARCHIVE_AFTER_DAYS=30）

## Flow→KO 交叉校验
- KO-01（冻结契约）← Memory Flow（session 起始冻结）+ Control Flow（主循环）
- KO-02（唯一承重边界）← Authority Flow（执行域分支）+ Policy Flow（SECURITY §2.2）
- KO-05（状态韧性族）← State Flow（WAL 回退/guard/readpool）
- KO-06（自我改进闭环）← Memory Flow（Curator 尾部）
- 全部 KO 均可回溯 ≥1 Flow Edge；无 Flow 与 KO 矛盾项。
