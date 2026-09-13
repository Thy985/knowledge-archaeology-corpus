# Independent Validation Report — ARCH-2026-09-14-001（guardian）

> Auditor 纪律：不引用考古包结论为事实；独立重读仓库（app.py 全 1412 行关键段 + init.sql + tests + workflows）建立 Independent Findings 后再对比。

## 判定统计

| 判定 | 数量 | 对象 |
|---|---|---|
| CONFIRMED | 11 | 7 检查编排无 LLM / Task token ACL（HS256+issuer）/ revocation 优先 / Gap 2 fix / 9 regex 族（parity 断言）/ sequence sandbox / hash integrity / adaptive rules（坏 regex 跳过）/ SHA-256 hash chain / canary 蜜罐 / hot path fail-safe |
| PARTIALLY_CONFIRMED | 2 | live 行为（hash chain DB 端）未 S5 实测；GitHub 元数据（API 限流未取） |
| DOWNGRADED | 0 | — |
| OVER_GENERALIZED | 0 | — |
| MISSING（Auditor 补充） | 2 | JWT 三级授权维度（granted_tools+tables+data_classes）与 escalation_policy 默认 deny（EK-02 补充）；缓存失败 stale-fallback 语义（EK-03 补充） |
| CONTRADICTED | 0 | — |
| NEEDS_HUMAN_REVIEW | 1 | 版本号不一致（pyproject 0.1.1 vs README "4.0.0"——需维护者确认语义） |

## 原 Archaeology 最重要的 3 个成功

1. **跨项目机制同构被正确识别并升级**（KO-02/03）：Guardian 的 SHA-256 hash chain + 写失败非致命，与 Aigis HMAC 链 + 日志异常容忍，是两个独立仓库的选择性同构——考古包将其从"单项目模式"升级为"2-project corroborated"且保留了升级证据链（无凭空升维）。
2. **fail-closed 边界刻画准确**（KO-03）：hot path fail-safe（misconfig/auth → halt）与 audit fail-open（写失败不阻断）的分层被独立验证——这是安全系统最重要的设计语义之一。
3. **修复史被保留**（EK-04 Gap 2 fix）："action 恒为 invoke 使 tool_id 路径不可达"的修复注释是考古包少数主动捕获的失败/修复证据，且测试（test_check2_halts_forbidden_tool_id）独立证实修复生效。

## 最重要的 3 个错误 / 缺口

1. **MISSING：JWT 授权维度被低估**（EK-02）：token 不止 granted_tools——还含 granted_tables（数据表级）与 granted_data_classes（数据类级）与 escalation_policy（默认 "deny" 升级策略）——"任务级最小权限"实际是**三级资源粒度**（工具/表/数据类），包内只写了工具级。（已 reconciliation 补）
2. **MISSING：缓存失败 fallback 语义未捕获**（EK-03 补充）：DB 查询失败 → **回退到 in-process 注册表（stale data）而非阻断**（"any DB/query error must fall back to stale data, not crash"）——这构成"撤销传播"的边界窗口：DB 不可达时已撤销工具可能短暂放行（内存态可能旧）。这是 fail-closed 语义的**已知例外**，包内未显式记录。（已 reconciliation 补）
3. **PARTIAL：检查 0 的 fail-open 窗口刻画略弱**（C-01）：无 token 跳过检查是显式设计（backward compat），但包内未量化"该窗口与 Phase 4 强制之间的安全暴露面"——已保留为假说，未虚构数字。

## 是否存在关键遗漏？
否（核心机制全覆盖）。补充项为增强：JWT 三级授权与缓存 fallback 语义均属"权限边界"维度的细化。

## 是否存在错误升维？
否。KO-01/02/03 的"2-project corroborated"升级有双仓库独立证据；KO-04（canary）正确保持单项目 pending；L5 方法论扣留正确。

## 是否存在事实错误？
否。抽查：9 族 regex（parity 断言 len==9 ✅）、JWT HS256+require exp/iat/jti/sub/iss+issuer 校验（app.py:300-311 ✅）、genesis 常量（app.py:851 ✅）、撤销优先（app.py:655-661 + 测试 ✅）、canary 位置（check 1 后，app.py:1250 ✅）。

## 是否存在 Flow 错误？
否。Control Flow（check 0-6 顺序）逐函数核对 app.py:1194-1362；Evidence Flow（prev_hash→INSERT PENDING→UPDATE row_hash）逐行对齐；Authority Flow（JWT scope + registry + capability + Bearer）真实。补充：缓存失败 fallback 分支应并入 4.3 Data Flow（stale data 路径）。

## Bypass / Fallback / Exception 路径专项
| 路径 | 行为 | 判定 |
|---|---|---|
| misconfig（TASK_TOKEN_SECRET 未设） | halt（GUARDIAN_MISCONFIGURED） | ✅ fail-closed |
| Bearer 缺失/无效 | halt（GUARDIAN_AUTH_FAILURE） | ✅ fail-closed |
| 无 task_token | 跳过 check 0（backward compat） | ⚠️ 设计态 fail-open（Phase 4 强制，C-01） |
| DB 缓存刷新失败 | fallback 到 in-process registry（stale） | ⚠️ 已知例外：已撤销工具可能短暂放行 |
| 坏 regex（adaptive） | skip + warning | ✅ fail-safe（规则不崩热路径） |
| 审计写失败 | warning + return -1（不阻断判定） | ✅ 审计 fail-open（设计声明） |
| JWT 解码意外异常 | 兜底 catch → None（不 masking） | ✅ fail-closed |
| canary 触发 | halt + CANARY_TRIGGERED | ✅ 蜜罐信号 |

## 是否发现新的 Benchmark / Regression Case？
是（3 个候选，供 skill CI / 品类基准参考）：
1. **撤销-缓存回归**：注册→批准→撤销→模拟 DB 失败（缓存 refresh 异常）→ 断言 check 1 是否仍 halt（验证 fallback 窗口是否可接受）。
2. **canary 探针回归**：对 guardian_canary 发起 /check → 断言 halt + CANARY_TRIGGERED 事件写入（蜜罐路径不回归）。
3. **Gap 2 回归**：action="invoke" + forbidden tool_id → 断言 halt（修复不回归——防再次退化为只查 action）。

## Reconciliation 决议
- 接受 Auditor 全部补充：EK-02 补 JWT 三级授权（tables/data_classes/escalation_policy=deny）；EK-03 补缓存失败 stale-fallback 语义（fail-closed 的已知例外）。
- 4.3 Data Flow 补 stale-data 分支注记。
- 不修改原包已写文件（保留初始考古记录）；补充作为 reconciliation 增量并入 Corpus 提交。
