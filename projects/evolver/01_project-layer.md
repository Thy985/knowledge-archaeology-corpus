# 01 Project Layer — Evolver 项目地图

> 全部事实来自 31b0691 快照实际内容（`repo/`）。混淆文件标注 🔒。

## 1. 项目身份

| 项 | 值 | 证据 |
|----|-----|------|
| name | @evomap/evolver | package.json |
| version | 1.94.0 | package.json |
| 定位 | "GEP-powered self-evolution engine for AI agents" | package.json description |
| license | GPL-3.0-or-later（2026-04-09 前为 MIT） | package.json + README Notice |
| 论文 | arXiv:2604.15097 "From Procedural Skills to Strategy Genes" | README |
| Node | >= 22.12 | package.json engines |
| 入口 | index.js（bin: evolver） | package.json bin |
| 测试 | 221 test 文件（node --test） | ls test/*.test.js |
| 混淆 | 57/202 src 文件（javascript-obfuscator ^5.4.1 devDep） | grep _0x + package.json |

## 2. 顶层结构

```
repo/
├── index.js（186KB 可读 CLI 主入口）/ cli-options.js
├── SKILL.md（自带 capability-evolver skill 定义：权限/网络/文件访问声明）
├── src/（202 js / 36,200 行）
│   ├── evolve/（🔒 8/8 混淆：主进化循环 + pipeline collect/dispatch/enrich/hub/select/signals + guards）
│   ├── gep/（44/86 混淆：GEP 协议——candidates/capsules/curriculum/epigenetics/executionTrace/feedbackEnvelope/solidify🔒/crypto🔒/contentHash🔒…；validator/ 4 文件可读）
│   ├── atp/（15 可读：Agent Transaction Protocol——hubClient/autoBuyer/merchantAgent/consumerAgent/protocol）
│   ├── proxy/（25/29 可读：模型代理——anthropic/bedrock/gemini/openai/ollama/vertex 路由 + trace/usage）
│   ├── adapters/（13 可读：claudeCode/codex/cursor/kiro/opencode 会话适配 + hook 脚本）
│   ├── experiment/（5 可读：agentRunner/comparison/metrics/triggerShift）
│   ├── ops/（9 可读）/ solo/（breaker+gitGuard）/ canary.js / forceUpdate.js / config.js / canonicalIdentityLock.js
│   └── webui/（30 可读：本地观察面板 client+observer+server）
├── test/（221 文件：安全/协议/一致性/回归/韧性）
├── scripts/（22：a2a_export/ingest/promote、skill2gep、gep_*、harness-governance-check、validate-*）
├── conformance/savings-core/（constants.json + golden-vectors.json 一致性基准）
├── assets/gep/genes.seed.json（种子基因）
└── README*.md（EN/zh-CN/ja-JP/ko-KR）/ LICENSE（GPL-3.0）
```

## 3. 六大模块面（含混淆状态）

| 面 | 文件 | 混淆 | 职责 |
|----|------|------|------|
| **进化引擎** | src/evolve.js + evolve/pipeline/* | 🔒 全混淆 | 主进化循环：扫描日志→选基因→生成 GEP prompt→记录事件 |
| **GEP 协议** | src/gep/*（86） | 44 混淆 | 基因/胶囊/候选/课程/表观遗传/执行追踪/反馈信封/资产存储/哈希/加密 |
| **安全验证** | src/gep/validator/*（4） | ✅ 可读 | sandboxExecutor 白名单执行 + reporter + stakeBootstrap |
| **ATP 经济层** | src/atp/*（15） | ✅ 可读 | Agent Transaction Protocol：hub 客户端/自动购买/商家/消费者 |
| **模型代理** | src/proxy/*（29） | 4 混淆 | 多模型统一路由（Anthropic/Bedrock/Gemini/OpenAI/Ollama/Vertex）+ trace/usage |
| **宿主适配** | src/adapters/*（13） | ✅ 可读 | Claude Code/Codex/Cursor/Kiro/OpenCode 会话钩子 |
| **自我更新** | src/forceUpdate.js + src/canary.js + index.js bootstrap | ✅ 可读 | 升级（备份/日志/原子重命名）+ 金丝雀 + 崩溃恢复 |
| **本地观察** | src/webui/*（30） | ✅ 可读 | 本地面板（运行/资产/管道/人设/安全观察） |

## 4. 核心数据结构

| 结构 | 定义 | 语义 |
|------|------|------|
| **Gene** | assets/gep/genes.seed.json | {type, id, category, signals_match[], preconditions[], strategy[], constraints[], validation[]}——协议化进化资产 |
| **Capsule** | GEP 资产（capsules.json） | 更高层进化资产（README GEP 段） |
| **EvolutionEvent** | events.jsonl | 可审计进化事件（README GEP 段） |
| GEP 资产存储 | ~/.evolver/gep/{genes,capsules,events}.json(l) | 本地运行时资产（git ignored，升级不覆盖） |
| 冲突验证结果 | solidify validation results | {ok, results:[{cmd, ok, err}]}（test 可执行契约） |
| 失败分类 | classifyFailureMode | {mode: soft/hard/none, reasonClass: validation/constraint_destructive/…, retryable} |

## 5. 核心状态

- **进化状态**：solidify_count（evolution_solidify_state.json）+ GEP 资产（genes/capsules/events）
- **守护进程状态**：锁文件（acquireLock）+ 心跳（lock lease mtime）+ 自适应 sleep 窗口（maxSleepMs=5min）+ OOM score adj
- **自我更新状态**：.evolver-force-update-backup-* 备份 + .evolver-force-update-journal.json 日志（中断恢复）
- **身份状态**：node_id（~/.evomap/node_id）+ canonicalIdentityLock + envFingerprint
- **代理状态**：proxy settings.json + trace usage 统计

## 6. 测试体系（221 文件——大型且主题化）

| 主题 | 代表测试 |
|------|---------|
| **安全** | sandboxExecutor.security（#451 shell 注入回归 + GHSA-jxh8-jh77-xh6g npm/npx 移除）、fetchSecurity（GHSA-r466-rxw4-3j9j --out= 路径逃逸、GHSA-cfcj-hqpf-hccf 默认分支）、crypto、rollbackSafety |
| **进化协议** | evolveGuards/evolveSelect/evolveDispatch/evolveSignals/evolveCollect/evolveEnrich/evolveHub/evolvePolicy |
| **自我更新** | forceUpdate*（7 文件：并发守卫/失败码/心跳/幂等/keep-list/中断报告）、lifecycle*（9 文件）、canary |
| **代理** | proxyAnthropic/proxyBedrock/proxyGemini/proxyOpenAI/proxyOllama/proxyVertex/proxyStreaming/proxyTokenReuse…（~20） |
| **ATP** | a2aProtocol（heartbeat/trace-guard）、atpExecute/atpTaskPickup/atpProxyRouting/atpAutoBuyer/atpAutoDeliver |
| **记忆** | memoryGraph/memoryGraphRotation/narrativeMemory/memoryFiltering/recallVerifier |
| **一致性** | savingsCoreConformance（golden-vectors）、schemaCapsule/schemaGene/schemaTask/schemaPromptConsistency |
| **韧性** | heartbeatResilienceRound3-9、syncEngineLoopResilience、loadBackoff |
| **源码级回归** | fetchSecurity（直接 grep index.js 源码模式）——**测试即静态分析** |

## 7. 主要配置（环境变量，README Configuration 段 + SKILL.md）

| 变量 | 默认 | 用途 |
|------|------|------|
| A2A_NODE_ID | 必填 | EvoMap 节点身份（注册后） |
| A2A_HUB_URL | https://evomap.ai | Hub API 基址 |
| EVOMAP_PROXY / EVOMAP_PROXY_PORT | 1 / 19820 | 本地代理（推荐） |
| EVOLVE_STRATEGY | balanced | balanced/innovate/harden/repair-only/early-stabilize/steady-state/auto |
| EVOLVE_ALLOW_SELF_MODIFY | false | 允许进化修改 evolver 源码（不推荐） |
| EVOLVER_ROLLBACK_MODE | stash | stash/hard/none（1.80.8 从 hard 翻转为 stash 防数据丢失） |
| GITHUB_TOKEN | 无 | GitHub API（自动 issue 上报/发布） |
| GEP_ASSETS_DIR | 无 | 覆盖 GEP 资产目录 |
| WORKER_ENABLED / 网站开关 | — | 工人池接入 |

## 8. 权限 / Policy / Governance 机制

| 机制 | 证据 |
|------|------|
| **SKILL.md 权限声明** | capability-evolver：execute 白名单 [git,node,npm] + network 白名单 [127.0.0.1, api.github.com, evomap.ai] + read/write 范围 + env_declarations（自查自约束） |
| **沙箱执行** | sandboxExecutor：ALLOWED_EXECUTABLES=['node'] + BLOCKED_NODE_FLAGS + 元字符拒绝 + 180s 超时 + cwd 限定 |
| **A2A 资产隔离** | a2a_ingest 暂存隔离区；a2a_promote 需 --validated + validation 审计 + 不覆盖同 ID 基因 |
| **fetch 路径遏制** | --out= path.resolve + path.relative 逃逸拒绝（GHSA-r466） |
| **进化约束** | GEP 协议约束：只允许 DNA emoji；不 improvisation（"Select an existing Gene by signals match (no improvisation)"） |
| **回滚治理** | EVOLVER_ROLLBACK_MODE：stash（可恢复）/hard（丢弃）/none；rollbackNewUntrackedFiles（gitOps） |
| **自我更新保护** | forceUpdate：结构化失败分类 + 备份 + 日志 + 安装标记 + "本地 GEP 资产永不覆盖" |

## 9. 主要外部依赖

- **运行时**：@evomap/gep-sdk ^1.5.0（GEP 协议 SDK）、@evomap/atp-sdk ^0.1.0（ATP 协议 SDK）、@aws-sdk/client-bedrock-runtime、dotenv、eventsource、undici
- **可选**：@napi-rs/keyring（系统钥匙串）
- **开发**：javascript-obfuscator（**生产代码混淆工具**）
