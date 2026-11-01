# Flow Atlas — Strands Box（七类流）

> 每类流从真实代码/设计锚导出，关键 Edge 可回溯 symbol/file/condition。

## 1. Control Flow（控制流）
```
run 启动（command/run.rs:80-103）
  ├─ 自硬化（拒同用户进程读地址空间）→ 读 box.toml + run 锁 + 记录 canonical 身份（decisions.md:177/191）
  ├─ telemetry::open（collector.rs:80）→ control(completed/refused)
  ├─ 装 policy（cap.rs judge：InertTemporalPermit → Err 拒 box）
  ├─ 解析凭据 + gateway 起（自签 CA）+ broker socket bind
  ├─ 构建沙箱 grants + 披露（stderr）→ 起 trampoline（自遏制→exec agent）
  └─ hosted()（run.rs:87）→ wait → teardown 反序 → collector.drained()（run.rs:103，export 存活于锁后）
```
请求路径：agent → alias(zsh/bash/sh/python3/python/MCP名) → run/box.sock（broker）→ {Shell | Monty | MCP broker} → policy 引擎（EffectInterceptor）→ 执行/拒绝 → 回 alias → agent。
出站路径：agent 沙箱内任意进程 → 代理变量 → gateway（net:connect 前+每地址 / http:request）→ policy → 转发 + 凭据注入。

## 2. State Flow（状态流）
| 状态 | 载体 | 转变 |
|---|---|---|
| box 生命周期 | run 进程 + run 锁（独占文件锁） | 启动 7 步；拆除反序；锁最后释放（第二个 run 拒绝） |
| collector 生命周期 | Collector queue（collector.rs:148 dropped/drained 双保险） | open → record/control → drained（唯一出口） |
| 政策集 | LoweredPolicySet（cap.rs） | 加载（验证/拒绝）→ 固定；deny-overrides 组合 |
| 事件历史 | per-box durable（decisions.md:775） | enforcement point 提交；temporal 规则读取；无回退非审计 |
| 沙箱 grants | 配置翻译产物 | 启动前一次构建、box 生命内固定（decisions.md:944） |
| CA | run 内存 | 每 run 新生；仅证书落 trust/cert.pem；私钥不出可信进程 |

## 3. Data Flow（数据流）
- 配置：box.toml → record/config/*（九键）→ grants 翻译 → trampoline 继承 descriptor（digest 绑定，decisions.md:1028）
- 请求：alias 连接 → broker 分腿 → enforcement point → policy（request details + history）→ verdict
- 凭据：vault（env:// 启动读 / aws:// 每请求签）→ gateway 注入 Mutation → 出站请求；response 回显 redact（attach.rs:183）
- 决策记录：enforcement point → collector → TargetExporter → records.jsonl（OTLP；decision 时间非 emit 时间，collector.rs:290）
- workload 遥测：agent 自己 spans → receive seam（loopback /v1/{traces,logs,metrics}）→ strip（剥离 box 保留属性命名空间）→ stamp → relay（receive.rs:42-71）

## 4. Evidence Flow（证据流）
- canonical record = **enforcement point 提交的有效决策**（含 reachable paths override），非引擎裁决（decisions.md:900-913）
- 决策记录默认落 workload 不可达路径（private/telemetry/records.jsonl）；effective-policy report 是诊断非 attestation（decisions.md:913）
- 无 approver 判定（decisions.md:809）：没有规则暂停等人；403 回执带 rule @id/@description 供 agent 自纠
- 证据强度：文档主张 vs 代码实现分开引用（本考古全部 Evidence 为 repo 实盘）

## 5. Authority Flow（权威流）
```
operator(box.toml + policy.dw)
  → run（可信进程，唯一权威持有者）
    → policy engine（唯一 authorizer，EffectInterceptor seam）
    → enforcement points（无自身裁决；CapabilityOutcome 无 Allow/Deny）
    → deny-only floors（网络前/interpreter 后，只能减）
  → alias/trampoline（零权力）
```
关键 Edge：egress-gateway 无 policy 依赖（decisions.md:570）；agent 内 harness 自带 approval 不是边界（decisions.md:111）。

## 6. Memory Flow（记忆流）
- policy history（per-box durable，跨 run；temporal 规则消费；无 tamper claim——decisions.md:761）
- interpreter 无跨调用状态（EK-B14）
- collector queue（drain 前缓冲；decision 时间戳=决策时刻防队列延迟失真）
- workload 遥测：strip（box 保留属性）→stamp（box 戳）→relay——agent 不能伪造/抑制记录（telemetry.md）

## 7. Policy Flow（策略流，治理闭环）
```
Decision（operator 写 policy.dw）
  → Approval（配置加载验证：解析/验证/不可触发拒绝）
  → Policy（dogwood 闭集动作词汇，deny-overrides，无编译 tier）
  → Enforcement（四 enforcement point + reachable floors；加载期 judge 拒绝惰性规则）
  → Future Decision（决策记录回灌：403+rule id 反馈 agent 自适应；records.jsonl 供 operator 复盘）
```
治理自约束闭环：AGENTS.md interface-frozen（改外部可观察面需用户批准，agent 不可自我授予）+ 9 premises 测试钉住 + tenets 优先序。

## Flow 指标
- 7 类流完整；关键 Edge 全部可回溯（symbol/file/condition 见各节）
- 与 KO 交叉校验：KO-B01←Authority+Policy Flow；KO-B03←Memory Flow；KO-B04←Control/Data Flow；KO-B05←State Flow
