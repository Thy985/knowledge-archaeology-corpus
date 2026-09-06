# 04 Flow Atlas — 七类流（从真实代码导出）

> 可读面（index.js/validator/atp/adapters/proxy/scripts）+ 测试契约导出；🔒=混淆实现（外部观察）。

## F1 Control Flow（控制流）

```
index.js main（命令分发 L890-3642）
  ├─ run//evolve/--loop：Preflight git 检查 → [loop: acquireLock → lock lease → OOM adj → EPIPE 保护]
  │     → evolve.js🔒（扫描日志→selector 选基因→生成 GEP prompt→EvolutionEvent）
  │     → loop: 自适应 sleep（interruptible）→ 心跳
  ├─ solidify：solidify({rollbackOnFailure})🔒 → 验证补丁 → 失败: Distiller 蒸馏修复基因
  │     → urgent questions → Hub（validationFailed 时）
  ├─ exec / distill / review / fetch（--out= 路径遏制）/ sync（--export=backup.gepx）
  ├─ webui / login / logout / setup-hooks / reset-local-secret
  ├─ atp-complete / buy / orders / verify / reuse / publish / recipe / experiment
  └─ 启动自检: _recoverInterruptedForceUpdateBootstrap（中断恢复，fail-closed）
```

**关键边证据**：命令分发 L890-3642（可读）；bootstrap 恢复 L1-80；solidify L2009；fetch 遏制（GHSA-r466 测试锁定）。

## F2 State Flow（状态流）

```
会话日志/信号（宿主 adapters 捕获）
  → GEP 资产状态: genes.json / capsules.json / events.jsonl（本地，git ignored）
  → 进化决策状态: selector JSON 决策（信号匹配）
  → solidify 状态: evolution_solidify_state.json（solidify_count）+ 补丁验证结果{ok, results[]}
  → 失败分类: soft（validation, retryable）/ hard（constraint_destructive, not retryable）
  → 学习回填: gene.signals_match += 学习信号（problem/area/action 结构化）
  → 升级状态: .evolver-force-update-backup-* + journal.json（中断可恢复）
  → daemon 状态: 锁 + lease mtime + 自适应 sleep 窗口
```

**关键边证据**：solidifyLearning.test.js（soft/hard 分类 + 回填）；forceUpdate.js 备份/日志；index.js isLoop（锁/心跳）。

## F3 Data Flow（数据流）

```
宿主会话（Claude Code/Codex/Cursor/Kiro/OpenCode）
  → adapters 钩子（session-start/end/signal-detect/task-recall）
  → memory/ 日志 + 信号 → evolve.js🔒 扫描
  → selector🔒 匹配 Gene/Capsule（signals_match → 分数排序）
  → GEP prompt（协议约束：DNA emoji only / 不 improvisation）
  → [solidify] Gene.validation[] 命令 → sandboxExecutor 白名单安检 → 执行（cwd=仓库根, 180s 超时）
  → 补丁结果 → 成功: 学习信号回填 + 失败: 回滚(stash) + Distiller 蒸馏修复基因
  → EvolutionEvent → events.jsonl（可审计）
  → Hub 通道: proxy（127.0.0.1:19820 mailbox）→ evomap.ai / api.github.com
```

**关键边证据**：sandboxExecutor.js（可读）；SKILL.md 网络端点声明（127.0.0.1 Proxy 强制）；scripts/a2a_*。

## F4 Evidence Flow（证据流）

```
进化证据链：
  EvolutionEvent（events.jsonl 追加）← 每轮进化决策记录
  solidify 验证结果（validation.results[{cmd, ok, err}]）← 补丁可验证
  trace/usage（proxy 层 token 用量）→ savings 测量（conformance 公式）
  executionTrace / capsuleExecutionTrace（GEP 执行追踪）
  antiAbuseTelemetry（防滥用遥测——test 存在）
外部审计面：
  webui observer（jsonl/runs/skills/personality/safety/redact）本地可视化
  recall-verify-report.js / harness-governance-check.js（脚本）
```

**关键边证据**：README EvolutionEvent；test/executionTrace 相关；scripts/recall-verify-report.js。

## F5 Authority Flow（权威流）

```
LLM（agent）权威：被约束——GEP 协议 prompt 生成"建议"，不直接改码（README: "prompt generator, not code patcher"）
代码权威：sandboxExecutor 白名单（node-only）最终裁决 validation 命令；BLOCKED_NODE_FLAGS 纵深
资产权威：本地 GEP 资产为选择唯一来源（selector 不 improvisation；种子基因仅首跑 seed）
用户权威：EVOLVE_ALLOW_SELF_MODIFY=false 默认（自我修改需显式开）；a2a_promote 需 --validated；EVOLVER_ROLLBACK_MODE 选择
Hub 权威：ATP 协议（verify/routing/proof/role/execution 枚举）约束经济层；发布/购买资产可拉回
升级权威：forceUpdate 安装标记验证（防误升级第三方仓库）+ fail-closed（恢复失败拒绝启动）
```

**关键边证据**：README "What Evolver Does"（L196-208）；sandboxExecutor；a2a_promote --validated（README L506）；forceUpdate _hasStrongEvolverInstallMarkers。

## F6 Memory Flow（记忆流）

```
短期（进程内）：GEP 资产加载 + selector 决策 + solidify 状态
长期（持久化）：
  ~/.evolver/gep/{genes,capsules,events}.json(l)——运行时资产（升级不覆盖）
  $EVOLVER_HOME/memory/*——memory graph / narrative / reflection
  记忆分层：memoryGraph + memoryGraphRotation（轮换防无限增长）+ narrativeMemory + memoryFiltering
  回忆：recallVerifier + recall-inject（任务时注入相关记忆）
外部：sync --scope=all --export=backup.gepx（Hub 拉回 + 便携备份）
```

**关键边证据**：SKILL.md file_access（writes: $EVOLVER_HOME/gep/* + memory/*）；test/memoryGraph*.test.js；README sync 命令。

## F7 Policy Flow（策略流）

```
决策（配置/环境变量）：EVOLVE_STRATEGY（balanced/innovate/harden/repair-only/early-stabilize/steady-state/auto）
  → 策略固化（执行点）：
      * selector 只选既有基因（不 improvisation）——策略写入 prompt 协议
      * validation 命令白名单（node-only）——安全策略
      * 回滚模式（stash 默认/hard/none）——数据保护策略
      * 自我修改开关（默认 false）——自指治理策略
      * 升级安装标记验证 + fail-closed——更新安全策略
  → 未来决策反馈：
      * 学习信号回填基因（成功/失败结构化学习）
      * urgent questions → Hub（失败时上报）
      * EvolutionEvent 审计日志
      * antiAbuseTelemetry（防滥用反馈）
```

**关键边证据**：SKILL.md env_declarations；README Configuration 段；solidifyLearning.test.js；index.js urgent questions 段（L2110-2145）。

---

## 流完整性检查（Flow→KO 交叉校验）

| 流 | 对应 KO | 关键边可回溯 |
|----|---------|-------------|
| F1 Control | KO-01/KO-03 | ✅ 命令分发 + solidify 链 |
| F2 State | KO-01/KO-06 | ✅ 测试契约 + forceUpdate |
| F3 Data | KO-01/KO-02 | ✅ sandboxExecutor + SKILL.md |
| F4 Evidence | KO-05/KO-06 | ✅ events.jsonl + trace + webui |
| F5 Authority | KO-02/KO-07 | ✅ 白名单 + 自我修改治理 |
| F6 Memory | KO-04 | ✅ GEP 资产 + memory graph |
| F7 Policy | KO-01/KO-04 | ✅ 策略变量 + 学习反馈 |

**盲区声明**：F1/F3 中 evolve.js🔒 内部边（扫描→选择的精确符号）不可读，以 CLI 调用点 + 测试契约为据；已标注，不冒充内部实现。
