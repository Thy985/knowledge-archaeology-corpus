# 06 — Validation & Evidence

## 验证方式
- 测试实测：`python3 -m pytest tests/ -q --tb=no` → **57 passed / 9 skipped / 1 warning（1.44s）**。
- 9 skipped 全部为 live 测试（tests/test_live.py 需 GUARDIAN_TEST_URL 真实 sidecar + PostgreSQL，跳过原因明确"run via 'make test-guardian-live'"）——**单元测试全绿，无失败**。
- 安装过程：pip install -e .（缺 respx 测试依赖 → 补装后 test_sdk 11 passed）。

## Truth Auditor（源真实）
| 声明 | 判定 | 证据 |
|---|---|---|
| 7 项确定性检查无 LLM | ✅ CONFIRMED | app.py:1194-1362 check 编排（纯函数 + 无模型调用） |
| Task token ACL（JWT scope） | ✅ CONFIRMED | app.py:287-341 + 611-651 |
| 撤销优先于批准 | ✅ CONFIRMED | app.py:653-661 + test_check1_revocation_takes_priority_over_approval |
| Gap 2 fix（tool_id 路径不可达修复） | ✅ CONFIRMED | app.py:681-712 注释 + test_check2_halts_forbidden_tool_id |
| SHA-256 hash chain 审计 | ✅ CONFIRMED | app.py:768-851 + init.sql audit_log |
| canary 蜜罐 | ✅ CONFIRMED | init.sql seed + app.py:1250-1280 |
| 版本号不一致 | ✅ CONFIRMED（事实本身） | pyproject 0.1.1 vs README "4.0.0" |
| 部署就绪性（S5 live） | ⚠️ PARTIALLY | 未跑 live 测试（环境依赖）；hash chain DB 行为以代码+init.sql 为准 |

## Coverage Auditor（覆盖）
- 已覆盖：7 检查全部实现 + 编排 + JWT + 缓存 + 审计链 + canary + 端点（check/health/metrics/rules）+ SDK + init.sql 全表 + CI 6 workflow + SECURITY.md。
- 未深读：CHANGELOG 版本史（浅克隆）、Makefile 细节、sdk/client.py 全部方法（176 行已略读，非核心）。
- 重要发现无遗漏：Gap 2 fix（修复史）、canary（独特机制）、写失败非致命（审计可用性）、版本不一致（可信度问题）全部捕获。

## Flow Auditor（流真实性）
- Control Flow（check 0-6 编排）逐函数核对 app.py:1194-1362——真实。
- Evidence Flow（audit_log 追加：prev_hash→INSERT PENDING→UPDATE row_hash）逐行对齐——真实；genesis 常量存在。
- Authority Flow（JWT scope + registry + capability 双路径 + Bearer）——真实。
- 未发现"想象出来的流"；4.6 Memory Flow 如实标注"Guardian 无记忆组件"（对照 Aigis 覆盖差异）。

## Abstraction Auditor（升维检查）
- KO-01/02/03 从"单项目"升级为"双项目互证"——基于 **Guardian 与 Aigis 两个独立仓库**的机制同构（hash chain / fail-closed / 确定性判定），升级有据（R1/R2/R3 簇内 EK 均 ≥2 且跨仓库）。
- KO-04（canary）保持单项目标注（Cross-project validation pending）——不因 KO-01-03 升级而连带升级。
- 无 L5 方法论（2 项目证据不足）——诚实扣留。

## Counterexample Hunter（反例）
- KO-01（结构空间判定）：反例 = check 3 的 9 regex 族仍处理**序列化 args 文本**（json.dumps）——"结构空间"不彻底，文本模式匹配仍在（M-01 张力）；已标注。
- KO-02（hash chain）：反例 = 审计写失败返回 -1（fail-open）——审计链本身可被静默缺失（不阻断但也不告警？warning 记录）——"链完整"与"链存在"是两回事；已标注。
- KO-03（fail-closed）：反例 = check 0 无 token 跳过（fail-open 窗口）+ check 3 log-only 放行（记录后允许）——fail-closed 仅覆盖"错误态"，不覆盖"设计态放行"；已标注。
- KO-04（蜜罐）：反例 = canary 依赖"合法 agent 永不调用"假设——若合法流程误配触发 canary，产生误报信号；单项目无实证。

## Epistemic Auditor（认知状态）
- Fact/Observation/Hypothesis/Pattern 区分合规：EK-01..14 为 Fact（S3/S4）；KO-01..04 为 Pattern/Model（双项目 S3+S4 或单项目 S3）；C-01..04 为 Hypothesis/Tentative。
- **Hypothesis 未冒充 Fact**：Phase 4 强制、novel sequence 误伤率、组合形态均为显式假说。
- **Cross-project 状态诚实**：KO-01/02/03 标 2-project corroborated（非"普遍定律"）；KO-04 标 pending。
- **矛盾显式化**：M-01（"No heuristics"表述 vs 正则规则）保留为未解决矛盾。

## 反例预算
- 4 个 L3+ KO 各 ≥2 定向反例（KO-01/02/03 各有 2+，KO-04 有 1 但单项目不适用预算）→ 反例预算消耗 7/12。

## 质量指标
| 指标 | 值 |
|---|---|
| EK 数量（links 覆盖率） | 14（100% 有 links；平均出边 >1；孤立 0） |
| KO 数量（聚合规则覆盖率） | 4（100% R1-R4；KO-01/02/03 双项目互证） |
| Candidates | 4（2 Hypothesis / 1 Cross-project / 1 Tentative） |
| 测试 | **57 passed / 9 skipped（1.44s）**；单元测试全绿 |
| skipped 归因 | live 测试需真实 sidecar 环境（GUARDIAN_TEST_URL），非产品失败 |
| 反例预算 | 7/12 消耗 |
