# Independent Validation Report — ARCH-2026-10-20-001（Strands Box）

> Auditor 盲重建：不引用考古结果，独立重读仓库建立 Findings 后对比。禁止修改原考古产物（修正经 Reconciliation 落地）。

## 判定统计（12 项独立抽查 + 8 项 Findings）

| 判定 | 数量 | 涉及 |
|---|---|---|
| CONFIRMED | 8 | EK-B01/B02/B03/B07/B08(核心)/B09/B10/B15/B16/B17/B19/B20（抽查覆盖） |
| PARTIALLY_CONFIRMED | 2 | EK-B05（judge 细节补充）、EK-B13（无状态为文档主张） |
| DOWNGRADED | 1 | EK-B08（凭据 scheme 文档-实现漂移） |
| OVER_GENERALIZED | 1 | EK-B12（home 拼写统一为防 forbid 逃逸，非仅"宽暴露面"） |
| MISSING | 1 | EK 未覆盖 QUARANTINE 治理机制（test-integ） |
| CONTRADICTED | 1 | credentials.md"两 scheme" vs 实现五源（记录为文档漂移非代码缺陷） |
| NEEDS_HUMAN_REVIEW | 1 | C-B01（组织级多 box 凭据隔离） |

## 独立抽查证据

| # | 抽查事实 | 独立证据 | 判定 |
|---|---|---|---|
| 1 | CapabilityOutcome 无 Allow/Deny | capability/traits.rs:13-19（Applied/Unavailable） | CONFIRMED |
| 2 | Unavailable→Deny{Credential}；collision→Deny | capability/decider.rs:12-35 | CONFIRMED |
| 3 | inject=always advisory override | credentials/attach.rs:48,130-136 | CONFIRMED |
| 4 | InertTemporalPermit 拒绝加载 | policy/cap.rs:64 + error.rs:34 | CONFIRMED |
| 5 | run 定序 + drained 唯一出口 + 非零退出≠BoxError | box/command/run.rs:78-103 | CONFIRMED |
| 6 | receive strip（box 保留属性） | telemetry/receive.rs:42-58（RESERVED_PREFIX strands.box. / RESERVED_SCOPE strands-box. / RENAMED_SCOPE agent-claimed:） | CONFIRMED |
| 7 | 代理变量名禁设 | box/record/config/env.rs:90（NODE_USE_ENV_PROXY 等） | CONFIRMED |
| 8 | trampoline 单向（macOS sandbox one-way permanent） | architecture.md:130-131 | CONFIRMED |
| 9 | 解释器路径拼写统一（home 缩写） | policy/adapters/script.rs:32-40（"Both interpreters must report the SAME spelling"） | CONFIRMED（S-3） |
| 10 | 凭据 scheme：文档两 vs 实现五 | credentials.md "only these two schemes" vs sources/{env,file,op,aws,aws_profile,credsd}.rs + backend.rs:10-29 | **DOWNGRADED（E-1）** |
| 11 | 遏制套件 QUARANTINE 治理 | test-integ/QUARANTINE.md（编译期 allowlist + 单测钉住 + SKIP 授权注释） | MISSING（E-3） |
| 12 | shell:spawn/shell:exec 决策 | policy/adapters/shell.rs:249,729,763 | CONFIRMED |

## 3 个成功（Auditor 支持性强化）

### S-1 · receive seam 的精确防伪造实现（CONFIRMED 强化）
- **发现**: 剥离前缀 `strands.box.`（agent payload）+ 作用域 `strands-box.` → 改名 `agent-claimed:`；agent 不能声明 box 保留属性（receive.rs:42-58）
- **影响**: EK-B07/B10 遥测族补"strip→stamp→relay"细节

### S-2 · 错误语义边界：workload 非零退出 ≠ box 失败（MISSING 补强）
- **发现**: run.rs:91-92 "A workload that exits non-zero is still `Ok`, because the box did its work; only a `BoxError` is the box's own failure"
- **影响**: 新增 EK-B21（box 自身失败与 workload 失败分离语义）

### S-3 · 统一路径拼写 = 防 forbid 逃逸（CONFIRMED 强化）
- **发现**: adapters/script.rs:32-40 —— Shell 报 `~/<relative>`，Python 若不缩写则同 `context.input.path` 对 permit fail-closed、对 **forbid fail-open**（forbid fs:read when path like "~/.ssh/*" 在 zsh 匹配、python3 不匹配）
- **影响**: EK-B12 补充"拼写统一是 forbid 防逃逸机制"（超越"宽暴露面"表述）

## 3 个错误/风险点（Auditor 主动攻击）

### E-1 · EK-B08 scheme 描述依赖过时设计文档（DOWNGRADED）
- **攻击**: credentials.md "Box accepts only these two schemes" vs 实现五源 `env:// file:// op:// aws:// credsd://`（sources/ 目录 + backend.rs:10-29）
- **修正**: EK-B08 标注"文档-实现漂移"；用户可见 box.toml 暴露面以 docs/user/egress.md 为准（待核验）；scheme 全集合以 sources/ 为事实

### E-2 · EK-B05 temporal 细节不足（PARTIALLY_CONFIRMED）
- **攻击**: cap.rs judge 不止"闭集上下文"，还有跨 permit 的覆盖率判定（无条件 permit 使 temporal permit 惰性→拒绝）；EK-B20 有拒绝结论但 B05 未挂 finding 机制
- **修正**: EK-B05 补 judge/finding 引用（cap.rs:22-74）

### E-3 · EK 缺 QUARANTINE 治理机制条目（MISSING）
- **攻击**: test-integ/QUARANTINE.md 是"编译期精确 allowlist + 单元测试钉住 + SKIP 只接受授权注释"的遏制测试治理模式（防静默跳过）
- **修正**: 新增 EK-B21b 或并入 EK 图（见 Reconciliation）

## 遗漏与 Coverage 声明
- 未独立深读：docs/user/tutorials/*、.agents/pocs、test-integ 各 cell 内部（黑盒未复跑）、op:///file:// 源内部实现
- **Benchmark case（供 skill CI Gold Record）**:
  - BC-BOX-1: "占位符换真凭据"——gateway 注入（env:///aws://）+ response redact + advisory override；agent 永不持真密钥（可测：box 内进程环境无真 secret）
  - BC-BOX-2: "加载期拒绝惰性规则"——InertTemporalPermit：无条件 permit 使 temporal permit 失效即拒绝加载（可测：policy load 失败而非静默失效）

## 结论
- 主 Package 核心机制全部确认（0 实质性 CONTRADICTED——E-1 为文档漂移非实现缺陷）
- 修正项：DOWNGRADED×1 + PARTIALLY_CONFIRMED×2 + MISSING×1 + OVER_GENERALIZED×1 → Reconciliation 落地
- EK 不删除；新增 EK-B21（错误语义边界）与 QUARANTINE 条目
