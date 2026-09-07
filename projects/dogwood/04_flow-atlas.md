# 04 · Flow Atlas — Dogwood（七类流）

> 从真实代码/文档导出；每条 Edge 标注可回溯 symbol / file。

## 04.1 Control Flow（控制流）

```
用户（agent 运行时 / CLI）提交事件 Event
  → Authorizer::is_authorized(event)（authorize/mod.rs:214，infallible）
      → 判断 decision point？──否──→ 返回 None（不判定）
      └──是──→ AuthorizerBuilder 注入的 PolicyEngine（CedarPolicyEngine 或远程）
                  → 时间条件求值 → TemporalEngine（InMemoryTemporalEngine，Mode 枚举）
                  → Provider 求值（extension/provider/eval.rs，确定性 Rhai 引擎）
                  → 组合裁决 → Response{decision, diagnostics}
  → 上层决定：允许（继续工具调用）/ 拒绝（阻止）
  → （生产）审计日志外包（README 明示缺失）
```
关键 symbol：`is_authorized`（Option<Response>）、`AuthorizerBuilder`、`Decision`、`allowed()`。

## 04.2 State Flow（状态流）

```
事件历史（event history）
  → InMemoryTemporalEngine（engine.rs:287，Mode 枚举；DB-backed 可插拔）
  → max_window 窗口（默认 24h，AGENTS.md）裁剪回看范围
  → pin 分区（PartitionKey，engine.rs:192）——重写 pass，"如同"按 pin 键分区（存储不分区）
策略状态：
  ParsedPolicySet（parse 两阶段）→ LoweredPolicySet（lower + distincter）
  → 决策点 gating：decision_kinds / is_decision_kind 匹配
```
关键 symbol：`PartitionKey`、`InMemoryTemporalEngine::Mode`、`is_decision_kind`、`LoweredPolicySet::from_str`。

## 04.3 Data Flow（数据流）

```
策略（.dw）→ parser（pest grammar）→ AST
  → extension/temporal 解析（parse.rs）→ 静态检查（check.rs：范围限制/嵌套聚合拒绝）
  → cedarify lowering（to_ast.rs + schema_augment.rs）→ Cedar 策略 + 增广 schema（context.<id> 槽）
事件（Event）→ 按事件 schema（.dwschema）解析（event_schema/parse.rs + derive.rs）
  → temporal 求值（from value）→ provider 输入（Value → Rhai Dynamic 转换）
Provider 数据：providers.json 声明 → Rhai 脚本（确定性引擎，OnceLock 共享）
  → 结果注入 context 槽 → Cedar 判定
```
关键 symbol：`LoweredPolicySet::from_str`、`PolicySchema::from_cedarschema_str`、`ServiceSchema::builder`、`fn engine()`（OnceLock）。

## 04.4 Evidence Flow（证据流）

```
决策证据：Response.diagnostics（authorize/mod.rs:39）
  → reason()（DogwoodRuleRef 迭代器——命中规则）+ errors()（str 迭代器）
可回放证据：trace_api.rs + corpus.rs + CLI replay（policy + trace → 逐事件裁决）
文档证据：docs-as-tests（dogwood-docs/tests/examples.rs 跑 CLI；guide 示例 bundle）
测试证据：boundary_*/cedarify_adversarial/expected_failures（拒绝路径 + 对抗 + 期望失败）
审计：README 明示"决策不记录日志"——审计是生产自补缺口（明示非静默）
```
关键 symbol：`Diagnostics::reason/errors`、`DogwoodRuleRef`、`dogwood replay`。

## 04.5 Authority Flow（权威流）

```
治理链（AGENTS.md 生命周期）：
  action schema（.cedarschema，手写或 MCP tools/list 生成）
    → service schema（.dwschema：decision/history points + pins + max_window + providers）
    → policies（.dw，autoformalize：自然语言 → 已验证策略）
    → CLI 验证门（dogwood validate/lower/replay）——每阶段强制
授权权威：
  permit/forbid + when/unless（Cedar 语义）
  → 时间条件（since/formerly/once/聚合）回看历史
  → provider 结果（注入 context）
  → 裁决：permit / deny /（错误 → deny，参考实现选择）
```
关键 symbol：`dogwood validate policy.dw --policy-schema schema.cedarschema`、`permit`/`forbid`、`when formerly within 1h`。

## 04.6 Memory Flow（记忆流）

```
事件历史 = 治理记忆：
  Event → event_schema 归一化（derive.rs）→ InMemoryTemporalEngine 存储
  → temporal 条件按 max_window 窗口检索（since/formerly/once/聚合）
  → 决策点事件（decision kinds）触发判定；历史点事件仅被回看
窗口边界：max_window（默认 24h）——远古事件不进入决策（资源 + 语义双重约束）
```
关键 symbol：`InMemoryTemporalEngine`、`max_window`、`formerly within 1h`、`Mode`。

## 04.7 Policy Flow（治理闭环：Decision→Approval→Policy→Enforcement→Future Decision）

```
Decision（设计）：
  Cedar 派生 + 时间条件（README）；formal spec（guide/08）
  Cargo.toml 单 crate 封装决策（设计注释）
Approval（治理门槛）：
  CONTRIBUTING.md / CODE_OF_CONDUCT / SECURITY.md；DCO（NOTICE 文件）
  AGENTS.md 生命周期 + .claude/skills 单源真相 + 强制 CLI 验证门
Policy（策略固化）：
  .cedarschema + .dwschema + providers.json + .dw 策略（语言本体）
  macros（宏库降低样板）
Enforcement（执行）：
  Authorizer + 可插拔 PolicyEngine/TemporalEngine
  → Validator::validate()（lowered 策略授权前强制）
  → deny-on-error（provider 出错，参考实现）
Future Decision（反馈）：
  CHANGELOG（2026-08-12 起，RichType 等演进）
  boundary/expected_failures 测试 → 语言边界修正
  生产限制清单（README）→ 部署决策输入
```
关键 symbol：`Validator::validate`、`dogwood validate`、`CHANGELOG.md`、`PolicySchema`/`ServiceSchema`。

## Flow→KO 交叉校验

| KO | 依赖 Flow 段 | 一致性 |
|----|-------------|--------|
| KO-01 过去维度 | 04.6（事件历史/窗口） | ✅ |
| KO-02 物化进 Cedar | 04.3（lowering 链）+ 04.7（Enforcement） | ✅ |
| KO-03 注入面 | 04.3（provider）+ 04.5（权威） | ✅ |
| KO-04 诚实边界 | 04.4（审计缺失明示）+ 04.7（限制清单） | ✅ |
| KO-05 自治理 | 04.7（Approval/Policy） | ✅ |
| KO-06 测试即契约 | 04.4（测试证据） | ✅ |
