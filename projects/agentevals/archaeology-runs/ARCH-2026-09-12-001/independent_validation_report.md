# Independent Validation Report — agentevals (ARCH-2026-09-12-001)

> Auditor 纪律：不引用 Archaeology Result 作为事实来源；独立重读仓库建立 Independent Findings 后对比。抽查方式：grep/symbol 级独立定位（builtin_metrics.py / converter.py / runner.py / loader/auto.py / storage/repos/* / run/service.py / tests/test_credential_injection.py）。

## 判定统计

| 判定 | 数量 | 涉及 |
|---|---|---|
| CONFIRMED | 14 | EK-01,02,03,04,05,06,07,08,09,10,12,14,15,16 + KO-01..04 主链 |
| PARTIALLY_CONFIRMED | 2 | EK-11（sink env headers 未深读）、EK-13（streaming no-op 门控细节未验） |
| MISSING | 1 | **worker lease 租约机制**（考古包未写） |
| DOWNGRADED | 0 | 无过度升维（L5 未写，守纪律） |
| OVER_GENERALIZED | 0 | 无跨项目 Principle 冒充 |
| CONTRADICTED | 0 | 无事实矛盾 |
| NEEDS_HUMAN_REVIEW | 1 | C-01（ADK 匹配算法为外部依赖，需 ADK 源码） |

## 独立抽查证据（Auditor 自己的 Findings）

1. **trajectory 归属**：builtin_metrics.py:10-27 直接 `from google.adk.evaluation...`——确认"复用 ADK 评估原语"（EK-06/KO-03）。
2. **提取器分派**：converter.py:61-77 `_detect_trace_format → get_extractor(trace).format_name()`——确认协议分派 + ADK 优先（EK-05）。
3. **credential fail-closed**：tests/test_credential_injection.py:4-6 明确 "fails loudly if the ADK judge seam moves" + test_unresolved_credential_errors_instead_of_ambient_auth——确认 EK-07，**且发现项目自带的 seam guard 测试**（考古包遗漏点之一）。
4. **run 状态机**：repos/__init__.py:50 "Extend the lease. Returns False if the run was cancelled or lost." + memory.py:85-90 `claim_next(worker_id, lease, max_attempts)`（QUEUED + attempt<max_attempts + created_at 排序）+ memory.py:104 `return not run.cancel_requested`——**确认 lease 租约 + cooperative cancel（worker 下次 heartbeat 观察 cancel_requested）**。考古包 04-S1/S3 写了 cancel 语义但**漏了 lease 租约机制**（MISSING-1）。
5. **双 semaphore**：runner.py:103-104 确认（EK-10）。
6. **loader 自检**：auto.py:45-48 确认（.jsonl / resourceSpans / batches / data 键）——EK-02。
7. **no golden → error**：builtin_metrics.py:34-36 + 381-384 确认（EK-14）。
8. **409 spec-mismatch**：run/service.py:28-35 + 53-54 确认（EK-09）。

## 最重要的 3 个成功（Archaeology 做对的）

1. **KO-01（评测/执行解耦）定位精准**：runner 纯函数消费 + loader 归一化层的因果链（EK-01→02→03→04）成立，且被独立抽查 A1/A7 证实。
2. **EK-07（fail-closed 凭据注入）** 捕捉到了本仓库最值得迁移的安全机制，且诚实标注了 ADK 私有 seam 技术债（TODO upstream 自证）。
3. **边界声明（EK-15）** 被提升为工程知识而非忽略——Claude Code/Codex/OpenCode 遥测缺口是与已考古 Corpus（OpenCode/dsh）连接的关键。

## 最重要的 3 个错误 / 遗漏

1. **MISSING：worker lease 租约机制未入包**。`claim_next(lease=timedelta)` + heartbeat 租约续期 + "cancelled or lost" 语义——这是 run 生命周期的并发控制核心（多 worker 场景防双重领取），比"双 semaphore"更接近基础设施层。**修正**：补入 04-S1/S2 与 02（EK-09 扩充）。
2. **PARTIALLY：EK-13 streaming no-op 门控**写为事实但未验证 `streaming=False` 的实际分支；应为 Observation（docstring 自述）。
3. **轻微：04-S3 cancel 描述缺"cancel_requested 标志由 worker 在下次 heartbeat 观察"的精确机制**——已通过独立抽查 memory.py:104 澄清（cooperative cancellation）。

## 关键遗漏检查

- 是否存在原 Archaeology 漏掉的重要 Engineering Knowledge？→ **lease 租约**（已补）；judge seam guard 测试（已补进 EK-07）。
- 是否存在错误升维？→ 否（L5 未写；KO 均 L3/L4 + 跨项目 pending）。
- 是否存在事实错误？→ 否（CONTRADICTED=0）。
- 是否存在 Flow 错误？→ 04 的 S3 cancel 语义已被澄清（cooperative），无虚构 Edge。
- 是否发现新 Benchmark / Regression Case？→ 是（B-1/B-2/B-3，见下）。

## 新增 Benchmark / Regression Case

- **B-1（benchmark）**：`_enrich_app_details` 合成 schema vs 真实 schema 的 multi-turn 指标评分差异（量化合成补全的偏差）。
- **B-2（benchmark/regression）**：零 tool span 的 trace 跑 tool_trajectory_avg_score 的行为——验证"遥测缺失"是否有显式信号（对比凭据 fail-closed；C-04 验证路径）。
- **B-3（regression，已在 repo）**：`test_credential_injection.py` 的 seam guard（ADK judge seam 移动时 fail loudly）——可作为知识考古 skill 的"验证器对实现 seam 变更敏感"参照。

## Reconciliation 决定

- 修正 Corpus Artifact（不改生产 Skill）：02 EK-09 扩充 lease 语义；04 S1/S2/S3 补租约与 cooperative cancel 精确机制；EK-07 补 seam guard 测试证据。
- Candidates C-01 标 NEEDS_HUMAN_REVIEW（外部依赖 ADK 匹配算法）。
