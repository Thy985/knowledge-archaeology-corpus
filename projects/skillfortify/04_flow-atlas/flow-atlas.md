# 04 Flow Atlas — 七类流

> 每条 Flow 的 Edge 均可回溯到实际 symbol / file / condition / state transition（dbb5942）。**不是架构想象**——关键边已直接读取或实测。

## 1. Control Flow（控制流）

**主扫描链**：
```
CLI: cli/main.py cli() group
  → cli/scan.py scan_command
    → discovery/system_scanner.py SystemScanner.scan_system()   # 系统级发现
    → discovery/ide_registry.py IDEProfile                       # IDE 配置档案
    → parsers/*.py SkillParser.can_parse()→parse()               # probe→parse 两阶段
      → parsers/base.py ParsedSkill                              # 统一中间表示
        → core/analyzer/engine.py StaticAnalyzer.analyze()       # 三阶段分析
          → Phase1 _infer_capabilities()                         # 能力推断
          → Phase2 _detect_dangerous_patterns()                  # 危险模式
          → Phase3 _check_capability_violations()                # 声明vs实际
          → core/analyzer/models.py AnalysisResult               # 结果
        → cli/output.py 渲染 / cli/dashboard_cmd.py → dashboard/generator.py  HTML
```

**并行命令链**：
- `verify`：cli/verify.py verify_command → StaticAnalyzer（单 skill）
- `lock`：cli/lock.py → core/lockfile/*（解析→锁定→哈希）
- `trust`：cli/trust_cmd.py → core/trust/engine.py TrustEngine
- `sbom`：cli/sbom_cmd.py → core/sbom/generator.py ASBOMGenerator
- `registry-scan`：cli/registry_cmd.py →（可选 httpx）registry 级扫描
- `frameworks`：cli/frameworks_cmd.py → 22 框架清单
- `dashboard`：cli/dashboard_cmd.py → dashboard/generator.py

**Edge 证据**：cli/main.py `cli.add_command(...)`（L56-63）；scan.py → system_scanner.py 调用链；parser `can_parse` 快速探测的"基于文件/目录存在"注释（parsers/base.py L92-97）。

## 2. State Flow（状态流）

**能力状态机（CapabilitySet）**：
```
Skill 内容 → CapabilitySet（per-resource 单 Capability，取 lattice join=最高级别）
  Edge: capabilities/models.py CapabilitySet.add() —— 始终保留最高 access
  状态转移: 遇到 network:READ 再加 network:WRITE → 状态=WRITE（join 保持）
  Edge 证据: models.py L96-103 "always keeping the highest access level (the lattice join)"
```

**信任状态机（TrustScore）**：
```
TrustSignals(四信号) → compute_intrinsic() → intrinsic ∈ [0,1]
  → compute_score(dependency_scores) → effective = intrinsic × min(deps)
  → score_to_level() → TrustLevel(L0-L3)
  → apply_decay() 随时间降 historical 信号（69 天减半 / 230 天 10%）
  → update_with_evidence() 正证据单调增，clamp 到 [0,1]
  Edge 证据: trust/engine.py L102-175,181-233; trust/propagation.py; tests/core/trust
```

**DY 攻击者知识集（K）**：
```
K = ∅ → intercept/inject/decompose/synthesize/replay 不断 add
  不变量: Monotonicity K(t)⊆K(t+1)；Interception/Synthesis closure
  状态转移拒绝: synthesize/replay 对不在 K 的消息抛 ValueError（DY closure violation）
  Edge 证据: threat_model/dy_skill.py（synthesize L127-139, replay L189-198 的 raise 分支）
```

**Lockfile 状态**：
```
skill 集合 → Lockfile.add_skill() → compute_integrity(SHA-256) → 持久化
  时间戳: _generation_timestamp() 由 SOURCE_DATE_EPOCH 锚定（可重现）
  Edge 证据: lockfile/lockfile.py L43-52,133-137
```

## 3. Data Flow（数据流）

```
22 框架源格式（.claude/SKILL.md / mcp.json / .openclaw / py imports...）
  → ParsedSkill（统一 IR：name/version/description/instructions/declared_capabilities/
     dependencies/code_blocks/urls/env_vars_referenced/shell_commands/raw_content）
  → StaticAnalyzer.analyze()
      ├─ _infer_capabilities → CapabilitySet（URL→network / shell→shell:WRITE /
      │    env→environment:READ / file→filesystem）
      └─ _detect_dangerous_patterns → list[Finding]（patterns.py 模式目录）
  → _check_capability_violations → violations
  → AnalysisResult(is_safe, findings, inferred_capabilities)
  → CLI 输出 / dashboard HTML / ASBOM
Edge 证据: parsers/base.py ParsedSkill 字段（L40-79）; analyzer/engine.py analyze()（L107-140）
```

## 4. Evidence Flow（证据流 / 安全判定流）

```
证据1: declared_capabilities（skill 自声明，可能撒谎）
证据2: inferred_capabilities（静态推断，保守过度近似=sound 证据）
   → CapabilitySet.violations_against(declared)（POLA 机械核对）
   → Finding(attack_class="privilege_escalation", Severity CRITICAL/HIGH)
   → is_safe = False
Edge 证据: capabilities/models.py violations_against（L107+）;
          实测: declared=filesystem:read + shell rm -rf → 2 findings（priv_esc CRITICAL+HIGH）
关键性质: 危险模式证据（Phase2）是召回型（可漏报）；能力证据（Phase1/3）是 sound（不漏报）
Edge 证据: engine.py L127-137 注释明确二者分离
```

## 5. Authority Flow（权威流）

```
权威授予: skill 声明 declared_capabilities → Capability.permits(required) 判定可执行
权威边界: CapabilitySet 每资源最高级别（join）→ 组合不放大（≤max 分量）
权威违反: inferred > declared → privilege_escalation（越权）
DY 侧权威: DYSkillAttacker 拥有网络/发布权威，但 synthesize/replay 受知识集 K 约束
  （不能从未知组件合成——DY closure violation 拒绝）
Edge 证据: capabilities/models.py permits; dy_skill.py synthesize/replay raise;
          实测 join(READ,WRITE)=WRITE（不放大到 ADMIN）
```

## 6. Memory Flow（记忆流 / 状态记忆）

```
持久记忆: Lockfile（磁盘上的 skill→哈希 映射，VCS 可重现）
运行记忆: DYSkillAttacker.knowledge（K 集合，单调不减，会话内）
生成记忆: BenchmarkGenerator._generation_order（确定性迭代，seed=42）
分析记忆: StaticAnalyzer 无跨调用记忆（每次 analyze 独立）——Stateless 分析器
Edge 证据: lockfile/lockfile.py; dy_skill.py __init__（self.knowledge=set()）;
          benchmarks/generator/core.py L394; 实测 analyze() 无缓存
```

## 7. Policy Flow（策略流 / 治理闭环）

```
POLA 策略（Decision）: 最小权限原则（Formal-Foundations §POLA）
  → 策略固化（Policy）: CapabilitySet.violations_against 作为可执行检查（models.py）
  → 强制执行（Enforcement）: StaticAnalyzer.Phase3 → Finding(privilege_escalation)
  → 影响未来决策（Future Decision）: AnalysisResult.is_safe 进入信任/锁文件判定

漏洞披露策略（SECURITY.md）:
  in-scope 决策 → 邮件报告路径 → 48h 确认 → coordinated disclosure
  范围外: 超大输入 DoS（known limitation）不处理

编辑治理策略（AGENTS.md/GitNexus）:
  编辑意图 → gitnexus_impact（上游影响分析）→ HIGH/CRITICAL 警告 → 才允许改
  → gitnexus_detect_changes（提交前变更核对）
Edge 证据: capabilities/models.py; analyzer/engine.py Phase3; SECURITY.md in/out scope;
          AGENTS.md "MUST run impact analysis before editing any symbol"
```

## Flow→KO 交叉校验门

| Flow Edge | 回溯 KO |
|-----------|---------|
| 过度近似推断（Evidence Flow §4） | KO-01 |
| violations_against → priv_esc（Authority/Policy Flow） | KO-02 |
| join 不放大（Authority Flow §5） | KO-03 |
| intrinsic→min→decay→monotonic（State/Trust Flow） | KO-04 |
| DY closure violation 拒绝（State/Memory Flow） | KO-05 |
| SHA-256 + SOURCE_DATE_EPOCH（State/Memory Flow） | KO-06 |
| normalize→分级（Evidence Flow） | KO-07 |
| seed=42 + leakage 检查（Memory/Policy Flow） | KO-08 |
| 文档声称 vs 实现（Policy Flow §治理闭环） | KO-09 |

## Reconciliation 补充：Authority Flow 的核对失效路径（EK-37）

```
声明过宽（ADMIN>=3 或通配）→ CapabilitySet 无法有效约束
  → Phase3 over-declaration 分支（engine.py L376-392）
  → Finding(HIGH, A6, "least-privilege checking cannot constrain this skill")
  → is_safe = False（显式报告，非静默放行）
无法解析声明 → unparsed 列表（engine.py L399+）
  → Finding(LOW, capability_violation, "these grant nothing")
  → 不静默丢弃（fail-safe）
Edge 证据: analyzer/engine.py L67, L376-392, L399+（定向实测确认）
```
