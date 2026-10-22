# 04 Flow Atlas — strands-box（ARCH-2026-10-23-001）

> 七类流从真实代码导出；Edge 标注可回溯（symbol / file / condition / state transition）。

## Control Flow（控制流）
```
box run <config> → cli_main（argv 解析 + run 硬化）→ prepare：LockedRun + acquire_run_lock
  → hosted：启动 containment（Seatbelt/namespace）+ 三适配器（egress/shell/script）+ MCP broker + Vault
  → agent 进程运行（每操作经 PEP→PDP）
  → 退出 → 释放 run lock
policy generate-schema（第二冻结命令）：schema 生成
```
- **Edge**: 三适配器共享同一 `Arc<PolicyEngine>`（box/AGENTS.md）；`shell:exec` vs `shell:spawn` 二选一（request.rs:142-240）

## State Flow（状态流）
- PolicyEngine：`unopened → open（加载+validate）→ decide 循环 → reload（live authority，in-flight 串行化）`（policy.rs:228-245）
- SchemaStage：分段装载（policy.rs:42）
- containment：`contained → operation 决策 → 终止`；contain_trampoline 衔接后端
- Vault：`preflight（credsd_preflight）→ load → attach（出站注入）`（credentials/src/preflight.rs）
- telemetry lane：`open → Collector 每决策记录 → records.jsonl`（一 lane/box）

## Data Flow（数据流）
```
box.toml → ConfigureRequest（验证一次，之后组合是装配——"Do not re-derive meaning from a string downstream"）
policy.dw → PolicyEngine（Policy{origin,text} → EffectivePolicy{engine,policy_id}）
PEP → Request（统一 enum）→ PDP.decide → Decision{Allow/Deny} → DecisionObserver（action/resource/decision）
  → telemetry records.jsonl
egress：CapabilityOutcome{Applied/Unavailable} → decide_request 折叠 → Verdict{Allow{mutations}/Deny{Credential|MutationCollision}}
credentials：Secret（opaque）→ Vault 解析 → Attachment（Inbound/Outbound/Redactions）→ 出站请求注入
```
- **Edge**: `resource()` 是标签非权威（request.rs:190-210）——"Nothing may resolve it back into a path and act on it"

## Evidence Flow（证据流）
- fail_closed_guards.rs：加载失败是契约——"a guard that lets one of these through has reopened the fail-open class"（测试=行为权威）
- test-workload 20 套件驱动 **built** binary（黑盒：baseline/git/mcp-stdio/kill-claude/resume-codex…）
- test-integ 显式 exclude（"excluded deliberately, not overlooked"——确定性遏制套件）
- verdict-hermetic / containment-deterministic / native-filesystem CI（11 workflows）
- 性能断言诚实标注：policy <1ms "asserted, not yet benchmarked"

## Authority Flow（权威流）
```
operator（author）→ box.toml + policy.dw（唯一 authorizer）
  → PolicyEngine（唯一决策权威，dogwood 引擎 pin 不可换）
  → PEP 三适配器（执行权威在 agent 进程外）
  → Decision（permit 显式 / forbid 覆盖 / 默认拒绝）
  → 回执（403 + rule id/description）或执行
reload：调用方负责认证（"Nothing here authenticates a request: the contained agent must never reach a reload"）
```
- **Edge**: Agent hostile by assumption（tenet 1）——"an agent earns that assumption: it runs model output, and it reads content it did not write"

## Memory Flow（记忆流）
- dogwood-local-engine durable history：事件先追加后评估 + 从私有 history 文件恢复 monitor 状态（temporal 前提）
- PolicyEngine 状态：reload 串行化在飞决策（authority 内建锁）
- Vault 状态：解析后 Secret 为 opaque（"an unusable secret value is unrepresentable"——决策 anchor）
- telemetry：records.jsonl（不写诊断日志文件——box/AGENTS.md "The box writes no diagnostic log file"）

## Policy Flow（治理闭环）
```
决策（tenets 7 条 + decisions.md 145 ADR + interface-frozen 批准门）
  → 策略（policy.dw authored Dogwood 规则）
  → 执行（三适配器 + containment + Vault 注入）
  → 记录（DecisionObserver → telemetry）
  → 未来决策（reload live authority #122；decisions.md 条目可修正——"correcting it is welcome"）
```
- **Edge**: MCP 双门（decisions.md#mcp-authorization-is-two-gates）——gate1 粗 permit、gate2 精化**非默认拒绝**（设计边界）；标准自述降格（decisions.md historical record）是治理闭环的元规则
