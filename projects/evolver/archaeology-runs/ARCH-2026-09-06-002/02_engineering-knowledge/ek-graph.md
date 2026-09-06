# 02 Engineering Knowledge — EK Graph（宽底座）

> 30 条 EK。🔒=源文件混淆（结论来自外部观察：CLI 调用点/README/test 可执行契约）。证据均可回溯 31b0691。

## EK-01 自进化循环（run/--loop）：扫描→选基因→生成 GEP prompt→记录事件
- **内容**：每轮进化：①扫描 memory/ 目录运行时日志/错误模式/信号 ②从本地 GEP 资产库选最佳匹配 Gene/Capsule ③生成协议约束的 GEP prompt（selector 输出 JSON 决策）④记录可审计 EvolutionEvent。
- **证据**：README "What Evolver Does"（L196-208）；GEP 段（L394-420）；evolve.js🔒（index.js 调用点）
- **epistemic**: Fact（文档+CLI 调用点；内部逻辑混淆，标 🔒）
- **links**: [causal: EK-01→EK-02（循环→solidify）], [subsystem: EK-01/EK-04 同进化引擎面]

## EK-02 solidify：验证补丁 + 失败分类 + 学习闭环
- **内容**：solidify 执行 Gene validation 命令验证补丁；classifyFailureMode 区分 soft（validation-only，可重试）/ hard（constraint_destructive 如 CRITICAL_FILE_DELETED，不可重试）；失败后 Distiller 自动蒸馏"修复基因"；adaptGeneFromLearning 把成功信号回填 gene.signals_match。
- **证据**：test/solidifyLearning.test.js（可执行契约）；index.js solidify 段（L2009-2145）；solidify.js🔒
- **epistemic**: Fact（测试可执行契约 + CLI）
- **links**: [causal: EK-02→EK-03（失败→rollback）], [mechanism: EK-05（学习信号）]

## EK-03 回滚治理：stash 优先 + 宿主仓库保护
- **内容**：EVOLVER_ROLLBACK_MODE=stash（git stash push --include-untracked，可 git stash pop 恢复）默认；hard（reset --hard 丢弃）仅显式选择；none 跳过；1.80.8 从 hard 翻转为 stash——防第三方宿主仓库数据丢失；isInsideEvolverRepo 防仓库身份混淆；rollbackNewUntrackedFiles 清理新文件。
- **证据**：SKILL.md（EVOLVER_ROLLBACK_MODE 注释）；test/rollbackSafety.test.js；gitOps.js（可读）
- **epistemic**: Fact
- **links**: [constraint: EK-03 约束 EK-02（失败→可恢复）], [mechanism: EK-06]

## EK-04 GEP 协议资产：genes/capsules/events 结构
- **内容**：Gene={type,id,category,signals_match,preconditions,strategy,constraints,validation}（种子基因）；Capsule 为更高层资产；EvolutionEvent 记录 events.jsonl；协议约束（只允许 DNA emoji、不 improvisation）。
- **证据**：assets/gep/genes.seed.json；README GEP 段
- **epistemic**: Fact（种子文件+文档）
- **links**: [causal: EK-04→EK-01（资产→选择）], [constraint: EK-04 约束 EK-08（本地资产保护）]

## EK-05 学习闭环：信号回填 + 失败蒸馏
- **内容**：solidify 成功 → 学习信号（problem:performance/action:optimize/area:orchestration）回填 gene.signals_match（结构化，action 类不过滤）；失败 → auto-distill 蒸馏修复基因（repair gene from failures）；urgent questions 上报 Hub。
- **证据**：test/solidifyLearning.test.js（adaptGeneFromLearning 断言）；index.js solidify 段（Distiller + urgent questions）
- **epistemic**: Fact（测试可执行契约）
- **links**: [causal: EK-05→EK-01（学习→后续选择）], [mechanism: EK-02]

## EK-06 沙箱执行（sandboxExecutor）：node-only 白名单 + 纵深
- **内容**：ALLOWED_EXECUTABLES=['node']（npm/npx 因 GHSA-jxh8-jh77-xh6g 移除——lifecycle-script RCE 类）；BLOCKED_NODE_FLAGS 深度防御（即使 node 也拒绝危险 flag）；parseCommand 拒绝全部 shell 元字符（; && | \` $() > < &）；空/非字符串拒绝；180s 超时；cwd 限定仓库根。
- **证据**：src/gep/validator/sandboxExecutor.js（可读，L44-271）；test/sandboxExecutor.security.test.js（#451 H1 shell injection via spawn({shell:true}) 回归）
- **epistemic**: Fact（实现精读 + 测试）
- **links**: [constraint: EK-06 约束 EK-02（validation 命令）], [contrast: EK-07]

## EK-07 文档-实现矛盾：README 白名单 vs 实现 node-only
- **内容**：README Security Model（L494-499）称 validation 命令前缀白名单为 node/npm/npx；实现 sandboxExecutor ALLOWED_EXECUTABLES=['node'] only——**文档与实现矛盾**（npm/npx 已被移除，README 未同步）。
- **证据**：README L494-499 vs sandboxExecutor.js L44 vs test 断言（npm/npx 不在白名单）
- **epistemic**: Fact（并列声明与实现）
- **links**: [contrast: EK-06], [causal: →Candidate C-02]

## EK-08 本地 GEP 资产升级保护（"永不覆盖"承诺）
- **内容**：.evolver/gep/ 资产 git ignored、升级不覆盖；assets/gep/ 为 bundled 种子（首跑复制不删原件；仅当无本地 genes.json 时 seed）；forceUpdate 对资产目录有保护逻辑。
- **证据**：README GEP 段（L406-419）；forceUpdate.js（安装标记/资产保护）
- **epistemic**: Fact
- **links**: [causal: EK-08→EK-16（资产→forceUpdate 保护）], [mechanism: EK-15]

## EK-09 forceUpdate：结构化失败分类 + 原子重命名 + 可恢复
- **内容**：forceUpdate 升级：结构化失败码（DEGIT_FAILED/DELETE_FAILED/COPY_FAILED/FALLBACK_*/DOWNLOAD_INCOMPLETE/NPX_NOT_FOUND）；备份前缀 .evolver-force-update-backup-* + 日志 .evolver-force-update-journal.json；安装标记验证（_isEvolverPackageName/_hasStrongEvolverInstallMarkers）；失败→崩溃恢复（bootstrap _recoverInterruptedForceUpdateBootstrap，fail-closed）。
- **证据**：src/forceUpdate.js（可读，L59-173）；index.js bootstrap（L1-80）；test/forceUpdate*.test.js（7 文件）
- **epistemic**: Fact
- **links**: [causal: EK-09→EK-10（升级→canary）], [mechanism: EK-16]

## EK-10 canary：重启前金丝雀
- **内容**：canary 在 daemon 用坏代码重启前捕获错误（"the canary catches it BEFORE the daemon restarts with broken code"）。
- **证据**：src/canary.js L6 注释；solidify 流程集成（index.js）
- **epistemic**: Fact（注释+调用点）
- **links**: [dependency: EK-10 依赖 EK-02（补丁验证）]

## EK-11 daemon 工程：EPIPE 保护 + OOM 调整 + 可中断 sleep
- **内容**：loop 模式：EPIPE 显式吞掉（daemon 必须比终端活得久）；OOM score adj -500（尽力提示）；interruptible sleep（SIGCONT 唤醒短路）+ ref'd timer（防 macOS 静默退出）。
- **证据**：index.js L890-1000（可读，注释详尽）
- **epistemic**: Fact
- **links**: [mechanism: EK-12（生命周期）]

## EK-12 生命周期与心跳：锁 + lease + 自适应 sleep
- **内容**：acquireLock 单实例；锁 lease 用 mtime 刷新（崩溃/PID 复用检测，不信任 kill(0)）；自适应 sleep（maxSleepMs=5min 默认）；preflight git 存在检查（#394 Windows 静默挂起修复）。
- **证据**：index.js（isLoop 段 L920-1000）
- **epistemic**: Fact
- **links**: [causal: EK-12→EK-11（生命周期→守护工程）]

## EK-13 测试即静态分析（源码级回归）
- **内容**：fetchSecurity 测试直接读 index.js 源码做模式匹配（outFlag.slice('--out='.length) → path.resolve(cwd) → rel.startsWith('..')）——GHSA 修复以源码模式锁定，防重构回归。
- **证据**：test/fetchSecurity.test.js（fs.readFileSync(index.js) + 正则断言）
- **epistemic**: Fact
- **links**: [mechanism: EK-06（都是安全治理）], [causal: →Candidate C-06]

## EK-14 conformance 一致性基准（golden-vectors）
- **内容**：conformance/savings-core/：constants.json + golden-vectors.json（spec_version 0.3.0）；r1_genebench_overall 公式 measured_savings（raw_tokens 489273/optimized 182943 → tokens_saved 306330/savings_pct 62.61 精确匹配）；savingsCoreConformance 测试通过。
- **证据**：golden-vectors.json；test/savingsCoreConformance.test.js（实测 PASS）
- **epistemic**: Fact（实测）
- **links**: [mechanism: EK-05（都是测量）], [causal: →Candidate C-07]

## EK-15 ATP 协议 SDK 提取（防枚举漂移）
- **内容**：ATP 枚举（VERIFY_MODES/ROUTING_MODES/PROOF_STATUSES/ROLES/EXECUTION_MODES）从 @evomap/atp-sdk 导入（thin CommonJS facade）；注释明确"v1.80.8 explore enum 事故"教训（手维护枚举值集 = 漂移）；SDK ESM/engines 兼容性处理（evolver 锁 >=22.12 使 require() ESM 可用）。
- **证据**：src/atp/protocol.js（可读全文）
- **epistemic**: Fact
- **links**: [causal: EK-15→EK-17（协议→hub）], [mechanism: EK-16]

## EK-16 依赖与 SDK 分层（gep-sdk/atp-sdk 独立发布）
- **内容**：GEP/ATP 协议核心逻辑进独立 SDK（@evomap/gep-sdk ^1.5.0 / @evomap/atp-sdk ^0.1.0），evolver 消费 SDK——协议与引擎解耦（多运行时共用，evox-Rust 未来接入）。
- **证据**：package.json dependencies；atp/protocol.js 注释
- **epistemic**: Fact
- **links**: [subsystem: EK-15/EK-16 同协议层]

## EK-17 进化网络：A2A/ATP hub 集成
- **内容**：EvoMap Hub 交互：a2a_export/ingest/promote 脚本；sync --scope=all --export=backup.gepx（拉回 Hub 购买/发布资产，便携备份）；autoBuyer/merchantAgent/consumerAgent；heartbeat 信号处理；event delivery。
- **证据**：scripts/a2a_*.js；src/atp/*；README Hub 段
- **epistemic**: Fact（可读面）
- **links**: [causal: EK-17→EK-18（hub→资产同步）]

## EK-18 资产同步与便携备份（.gepx）
- **内容**：sync 从 /a2a/assets/purchased + /published-by-me 拉回资产，重物化 genes.json/capsules.json，打包 .gepx；已购 payload 零成本重取；纯本地未上传资产无远程副本（README 明示恢复路径）。
- **证据**：README GEP 段（L410-418）
- **epistemic**: Fact
- **links**: [mechanism: EK-08（资产治理）]

## EK-19 A2A 资产隔离：ingest 暂存 + promote 审计
- **内容**：外部基因/胶囊 a2a_ingest 进隔离候选区；promote 需显式 --validated；Gene validation 命令按同安全检查审计（不安全→拒绝）；同 ID 基因不覆盖。
- **证据**：README Security Model（L504-511）
- **epistemic**: Fact
- **links**: [constraint: EK-19 约束 EK-17（外部资产入口）], [mechanism: EK-06]

## EK-20 模型代理面（proxy）：多模型统一路由
- **内容**：proxy 支持 Anthropic/Bedrock/Gemini/OpenAI Responses/Ollama/Vertex 路由；chat completions + responses + streaming + SSE；token 复用（proxyTokenReuse）；trace/usage 提取；settings 持久化。
- **证据**：src/proxy/router/*（可读）+ test/proxy*.test.js（~20 文件）
- **epistemic**: Fact
- **links**: [causal: EK-20→EK-21（代理→trace）]

## EK-21 trace/usage 追踪（可观测）
- **内容**：proxy/trace/extractor + usage（token 用量）；trajectory-export 命令；traceThinkingEffort/traceUserIdHash/traceRuntimeMetadata 测试。
- **证据**：src/proxy/trace/*；test/trajectory*.test.js
- **epistemic**: Fact
- **links**: [causal: EK-21→EK-14（用量→savings 测量）]

## EK-22 宿主适配（adapters）：会话钩子
- **内容**：Claude Code/Codex/Cursor/Kiro/OpenCode 适配器 + hook 脚本（evolver-session-start/end/signal-detect/task-recall）；sessions_spawn(...) 文本由宿主解释（OpenClaw 拾取 stdout 指令；独立模式只是文本）。
- **证据**：src/adapters/*；README "How It Integrates"（L210-222）
- **epistemic**: Fact
- **links**: [causal: EK-22→EK-01（会话信号→进化输入）]

## EK-23 身份与加密：node_id + canonicalIdentityLock + crypto🔒
- **内容**：node 身份（~/.evomap/node_id）+ canonicalIdentityLock（身份元组同步）+ crypto🔒（AES-CBC 加盐，从混淆模式推断）+ reset-local-secret 命令 + 可选 keyring。
- **证据**：src/canonicalIdentityLock.js（可读）；index.js reset-local-secret；crypto.js🔒（混淆）
- **epistemic**: Fact（部分：加密实现混淆，算法从模式推断=Observation）
- **links**: [mechanism: EK-23/EK-09（都是身份/安全治理）]

## EK-24 记忆系统（memory graph + narrative）
- **内容**：memoryGraph/memoryGraphRotation（图轮换）/narrativeMemory（叙事记忆）/memoryFiltering/recallVerifier + recall-inject——进化记忆分层。
- **证据**：test/memoryGraph.test.js 等（4 文件）+ src/gep 对应（部分混淆）
- **epistemic**: Fact（测试契约）
- **links**: [causal: EK-24→EK-01（记忆→信号扫描）]

## EK-25 自我修改边界：EVOLVE_ALLOW_SELF_MODIFY 默认 false
- **内容**：允许进化修改 evolver 源码的开关默认关闭（"NOT recommended"）；SKILL.md 文件访问声明 "src/** evolved code, only during solidify"——自我修改被显式治理而非默认允许。
- **证据**：SKILL.md env_declarations；README 配置段
- **epistemic**: Fact
- **links**: [constraint: EK-25 约束 EK-02（solidify 范围）], [causal: →Candidate C-05]

## EK-26 混淆与开源治理：57/202 文件混淆 + source-available 转向
- **内容**：src/evolve 全模块 + gep 半数 + proxy 4 文件被 javascript-obfuscator 混淆；README Notice 声明"从 fully open source 转向 source-available"（2026-03 Hermes Agent 相似性争议后）；GPL-3.0-or-later 声称 vs 混淆实现并存。
- **证据**：grep _0x 统计；README Notice（L20-30）；package.json devDeps（javascript-obfuscator）
- **epistemic**: Fact（混淆范围实测）；Interpretation（动机为保护性推断）
- **links**: [causal: EK-26→Candidate C-01（GPL 合规）], [mechanism: EK-27]

## EK-27 混淆不影响执行与测试（但影响审计）
- **内容**：混淆文件仍可 require 运行（solidify.js🔒 被 test require 并测 classifyFailureMode/adaptGeneFromBuilding 等）；可读 index.js 是测试静态分析的锚点（混淆文件无法被源码级回归锁定）。
- **证据**：test/solidifyLearning.test.js require 混淆的 solidify.js 成功；fetchSecurity 依赖 index.js 可读
- **epistemic**: Fact（实测）
- **links**: [constraint: EK-27 约束 EK-13（静态分析仅覆盖可读面）]

## EK-28 webui 本地观察面板
- **内容**：src/webui/：observer（jsonl 事件/runs/skills/personality/safety/redact）+ server + client（bootstrap/pipelines/overview + echarts）；本地运行可视化。
- **证据**：src/webui/*（30 可读）
- **epistemic**: Fact
- **links**: [mechanism: EK-21（可观测）]

## EK-29 skill 系统自指（Evolver 自带 SKILL.md）
- **内容**：仓库自带 capability-evolver SKILL.md（权限/网络/文件访问/环境声明）——一个自我进化的 skill 引擎以 skill 协议自我描述；skill2gep/skillDistiller/skillPublisher/skill2recipes 把 skill 转化为 GEP 资产。
- **证据**：SKILL.md 全文；scripts/skill2gep.js；src/gep/skillDistiller.js
- **epistemic**: Fact
- **links**: [mechanism: EK-04（资产转化）], [causal: →Candidate C-08]

## EK-30 韧性测试体系（heartbeatResilience Round3-9）
- **内容**：连续 7 轮心跳韧性测试（Round3-9）+ syncEngineLoopResilience + loadBackoff + cycleHardTimeout——系统化验证 daemon 在各类中断/限流/超时下的存活。
- **证据**：test/heartbeatResilience*.test.js（7 文件）+ 相关
- **epistemic**: Fact
- **links**: [mechanism: EK-11/EK-12（守护工程）]

---

## EK 边统计（防退化检查）

- 总 EK：30；含 links：30/30（100%）；游离：0；平均出边 ~1.8
- 🔒 标注：8（EK-01/02 部分、EK-23 加密部分）——凡涉及混淆实现均显式标注证据等级
