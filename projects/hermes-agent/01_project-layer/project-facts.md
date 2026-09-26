# Project Layer — Project Facts（hermes-agent v0.21.3 @ fba4cb1）

> 全部事实可回溯到快照仓库实际内容（commit `fba4cb1`）。

## F-01 仓库身份
NousResearch/hermes-agent，Python 个人 Agent，v0.21.3（pyproject.toml），13,988 文件，Apache/MIT 双许可说明见 LICENSE。（来源：pyproject.toml、git ls-files）

## F-02 多前端同核心
同一 agent 核心运行于 CLI / 消息网关（Telegram/Discord/Slack 等 ~20 平台）/ TUI / Electron desktop（apps/desktop）。（来源：AGENTS.md "What Hermes Is"）

## F-03 系统提示词冻结记忆
MEMORY.md（agent notes）+ USER.md（user profile）以 **FROZEN snapshot** 进入系统提示词（session 起始）；mid-session 写磁盘但不改 prompt（prefix cache 完好）。（来源：tools/memory_tool.py docstring）

## F-04 单一 memory 工具
`memory` 工具：add/replace/remove 或批量 operations 列表；MEMORY.md/USER.md 双目标。（来源：tools/memory_tool.py）

## F-05 记忆 § 分隔卡片
learning_graph 把 MEMORY.md/USER.md 按裸 `§` 分隔符拆 chunk → 每 chunk 一张卡（memory 卡先、profile 卡后）；引用形如 `memory:<source>:<index>`（source=memory|profile）。（来源：agent/learning_graph.py:130-134、agent/learning_mutations.py:4-5）

## F-06 Skills = 程序性记忆
SKILL.md 是 agent 的程序性记忆（narrow "how to do X"）；MEMORY.md/USER.md 是宽声明性记忆；布局 `<skills>/[category/]<skill>/SKILL.md` + references/templates/scripts/assets。（来源：tools/skill_manager_tool.py docstring）

## F-07 技能安装扫描
skills_guard.py：静态正则扫描外部来源技能 + 信任感知安装策略；SCANNER_VERSION="skills-guard-v5"。（来源：tools/skills_guard.py）

## F-08 信任分级
TRUSTED_REPOS={openai/skills, anthropics/skills, huggingface/skills, NVIDIA/skills}；INSTALL_POLICY 按 builtin/trusted/community/agent-created × safe/caution/dangerous 三档；agent-created dangerous → "ask"（报错给 agent 重试）。（来源：tools/skills_guard.py）

## F-09 技能硬验证
skill_manager_tool._validate_frontmatter 是硬验证（block 非协商项）；skill_linter 是咨询性 companion（Findings 不自行 block）。（来源：tools/skill_linter.py docstring）

## F-10 技能供应链已知缺口
静态正则无法把语言写 API（open/write_text/shutil.copy/fs.writeFileSync）与动态目标绑定——agent-config 文件写入只报低级 *_ref finding；未来需第四"mechanical" tier。（来源：tools/skills_guard.py docstring "Known gap"）

## F-11 Curator 空闲维护
agent/curator.py：空闲触发（无 cron daemon）后台技能维护；last run 超 interval_hours（默认 7 天）且空闲超 2h 时自动迁移生命周期状态，可选 fork AIAgent pin/archive/consolidate/patch skills；持久化 `.curator_state`。（来源：agent/curator.py）

## F-12 Curator 不变量
只动 curator-managed skills；永不删除只归档（可恢复）；pinned skills 绕过自动迁移；fork 用 auxiliary client 不碰主会话 prompt cache。（来源：agent/curator.py docstring）

## F-13 状态存储 SQLite
hermes_state.py：SQLite state store（session metadata/message history/model config/FTS5）；WAL 模式；compression 用 parent_session_id 链；session source-tagged（cli/telegram/...）。（来源：hermes_state.py docstring）

## F-14 WAL 兼容回退
hermes_state_wal.py：WAL 不兼容标记（NFS/SMB/CIFS/FUSE/WSL1 "locking protocol"、ZFS "-shm 损坏 disk i/o error"、FUSE "not authorized"）→ 回退 DELETE journal（读阻塞写）；WAL size limit 64MiB。（来源：hermes_state_wal.py）

## F-15 状态库 guard
hermes_state_guard.py：`_STATE_DB_GUARD_BYPASS_ENV`、`_is_production_state_db`、`_register_test_instance`——生产/测试实例隔离保护。（来源：hermes_state_guard.py）

## F-16 回滚权威
hermes_state_rewind.py：carrier-aware 用户轮回滚（/undo /retry）唯一实现（CLI/gateway/TUI 共用）；durable transcript 是权威，warm in-memory history 只需一致。（来源：hermes_state_rewind.py docstring）

## F-17 检查点快照
tools/checkpoint_manager.py：文件变更前（每目录每轮一次）透明快照到共享 shadow git store `~/.hermes/checkpoints/`（GIT_DIR/WORK_TREE/INDEX_FILE 隔离不污染用户项目；跨项目 blob 去重）。（来源：tools/checkpoint_manager.py docstring）

## F-18 注入扫描 scope
prompt_builder._scan_context_content：context 文件（AGENTS.md/.cursorrules/SOUL.md）注入扫描，命中即 BLOCK（`[BLOCKED: ...]` 不加载）；scope="context"（strict-scope SSH-backdoor/persistence/exfil 对克隆仓库文档太激进）。（来源：agent/prompt_builder.py:82-108）

## F-19 注入扫描 user_authored 例外
HERMES_HOME 内用户自己的 SOUL.md（user_authored=True）命中只 WARN 仍加载（#112570：用户写"ignore previous instructions"作为安全指南不能因一行日志丢整个身份文件）；distribution.yaml 拥有的 SOUL.md → user_authored=False（block）。（来源：agent/prompt_builder.py docstring）

## F-20 文件安全自我定位
agent/file_safety.py："Every guard here is defense-in-depth, NOT a security boundary"——terminal 与 OS 用户同权限；价值=对尊重工具错误的模型的明确拒绝 + 可见审计轨迹。（来源：agent/file_safety.py docstring）

## F-21 唯一承重边界
SECURITY.md §2.2："The only security boundary against an adversarial LLM is the OS-level isolation"（terminal 容器/云沙箱/远程后端）；in-process 启发式非边界；§3 超出边界的问题关闭（欢迎走普通 issue/PR）。（来源：SECURITY.md §2.2/§3）

## F-22 prompt cache 神圣
AGENTS.md 不变量 1："Per-conversation prompt caching is sacred"；唯一例外 context compression；mutating slash 命令默认 deferred 下 session + opt-in `--now`（/skills install --now 为范式）。（来源：AGENTS.md）

## F-23 narrow waist
AGENTS.md 不变量 2："core is a narrow waist"——每个 model tool 随 API call 发送；新能力走 CLI+skill/service-gated tool/plugin，不扩核心。（来源：AGENTS.md）

## F-24 工具注册自动发现
tools/registry.py 零依赖；每个 tools/*.py 顶层 `registry.register()`；model_tools.py 触发 discover_builtin_tools()；所有 handler 返回 JSON string。（来源：tools/AGENTS.md）

## F-25 记忆 provider 生命周期
memory_provider.py：插件 provider（一个外部 + 内置）生命周期 initialize→system_prompt_block/prefetch/sync_turn→tool dispatch→shutdown + on_* hooks；CHECKPOINT_API_VERSION=2（v2 opt-in fail-closed：normalized evidence handoff + strict-mode failure propagation）；v1 best-effort。（来源：agent/memory_provider.py）

## F-26 MemoryManager 扇出
memory_manager.py：把 memory hooks fan-out 到 providers；内置 provider 恒允许；**只允许一个外部插件 provider 同时注册**（tool-schema bloat、conflicting backends）。（来源：agent/memory_manager.py docstring）

## F-27 并发隔离
ctx_bound/spawn_context_thread：provider 后台任务（prefetch/sync/writer loops）绑定调用方 contextvars；空 context 的 worker 落到默认 profile 或 secrets fail-closed。（来源：agent/memory_provider.py）

## F-28 入口 bootstrap
cli.py 首 import 必为 hermes_bootstrap（UTF-8 stdio Windows 兼容）；os.environ["HERMES_QUIET"]="1" 抑制启动噪音。（来源：cli.py:1-14）

## F-29 可选依赖面
pyproject：35 直接依赖 + 45 optional-deps groups（anthropic/bedrock/azure/mcp/supermemory/mem0/honcho/feishu/dingtalk/...）。（来源：pyproject.toml）

## F-30 测试规模
tests/ 4,645 文件 + tests-js/；batch_runner.py/mini_swe_runner.py（RL/benchmark 轨迹）；evals/（memory/ 等）。（来源：git ls-files、仓库结构）

## F-31 轨迹压缩策略
trajectory_compressor.py：保护 head（system/human/first gpt/first tool）+ 最后 N 轮；中间只摘要必要轮数（**不拆 tool_call/tool_response 对**）。（来源：trajectory_compressor.py docstring）

## F-32 人格文件
SOUL.md 为风格/人格（"Be direct"、不废话、不用形容词堆砌），注入扫描范围内；非记忆文件。（来源：SOUL.md、prompt_builder.py:82）

## F-33 内存/记忆双轨
memory_tool_store.py：ENTRY_DELIMITER、MEMORY_BLOCK_HEADERS、MemoryStore、_scan_memory_content；记忆内容扫描在 store 层存在。（来源：tools/memory_tool_store.py）

## F-34 记忆指令审批
file_tools_write_guards.py：SOUL.md 与 config.yaml 同信任类——file-tool 写经 protected-instruction approval gate。（来源：agent/prompt_builder.py docstring 引述）

## F-35 上下文压缩提示
context_compressor.py：压缩后插入 "Your persistent memory (MEMORY.md, USER.md) in the system prompt is ALWAYS authoritative and active"。（来源：agent/context_compressor.py:4603）
