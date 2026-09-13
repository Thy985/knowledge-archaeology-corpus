# 04 — Flow Atlas（七类流，全部可回溯）

## 4.1 Control Flow（工具调用治理主链路）
```
Agent 调用工具
  → sidecar /check（FastAPI :9766）接收 GuardianCheckRequest
  → Bearer auth（GUARDIAN_REQUIRE_AUTH=true；misconfig/auth 失败 → halt，fail-safe）
  → _maybe_refresh_caches（注册表/规则缓存更新）
  → Check 0 Task token ACL → Check 1 Tool registry → Canary 检查 → Check 2 Capability
  → Check 3 Destructive（halt / log-only 分级）→ Check 4 Sequence（novel → sandbox）
  → Check 5 Hash integrity → Check 6 Adaptive rules
  → 决策：allow（继续执行）/ halt / sandbox / log-only（记录后放行）
  → 记录 threat_events（异步）+ audit_log hash chain
```
**Edge 证据**：app.py:1194-1362 check 端点逐行 + 各 _check_N 函数（611-1034）。

## 4.2 State Flow（工具与规则状态机）
```
a) tool_registry：PENDING → APPROVED / REVOKED（撤销优先于批准，10s 缓存传播）
b) threat_rules：PENDING → APPROVED / REJECTED / EXPIRED（60s 热加载）
c) 响应分级：allow / halt / sandbox / log-only（无状态机——每次调用独立判定）
```
**Edge 证据**：init.sql tool_registry status CHECK 约束 + threat_rules status CHECK + app.py check 3/4/6 分级分支。

## 4.3 Data Flow（调用 → 判定 → 审计）
```
GuardianCheckRequest（tool_id/args/action/task_token/agent_id/run_id）
  → args JSON 序列化 → detect_destructive_pattern（9 regex 族）
  → task_token → JWT 解码 → granted_tools scope 检查
  → tool_id → 注册表缓存（approved/revoked + description_hash/schema_hash）
  → adaptive rules（DB JSONB → 正则/名单/序列匹配）
  → 缓存刷新失败 → fallback in-process registry（stale data，非阻断）
  → 结果 → threat_events 表（JSONB metadata + raw_input 截断 500）
  → 审计：payload canonical（sort_keys）→ SHA-256(seq|ts|event_type|agent_id|payload|prev_hash) → row_hash
```
**Edge 证据**：app.py:583-611 request/response 模型 + 713-768 威胁事件写入 + 768-851 审计链。

## 4.4 Evidence Flow（审计链防篡改）
```
audit_log 追加：
  SELECT 最后一行 row_hash → prev_hash（无记录用 _AUDIT_LOG_GENESIS）
  INSERT（row_hash='PENDING'）→ 计算 canonical SHA-256 → UPDATE row_hash
  → 链：每行 prev_hash = 前行 row_hash；篡改任意字段 → 后续链断裂
  → 写失败 → warning + return -1（非致命，审计缺失不阻断判定）
```
**Edge 证据**：app.py:768-851（_compute_audit_row_hash_direct canonical 拼装 + _append_audit_log_direct）+ init.sql audit_log 表。

## 4.5 Authority Flow（权限边界）
```
a) Task token（JWT）：签名验证（TASK_TOKEN_SECRET）→ granted_tools 白名单（check 0）
b) tool_registry：approved 名单 + revoked 名单（check 1）——撤销优先
c) FORBIDDEN_CAPABILITIES 黑名单双路径：action 类型 + tool_id 名称（check 2）
d) Bearer auth（sidecar 网关层）：GUARDIAN_REQUIRE_AUTH=true 时强制
```
**Edge 证据**：app.py:287-341（JWT 验证）+ 653-712（registry/capability）+ 418-459（Bearer）。

## 4.6 Memory Flow（跨会话/记忆——Guardian 无记忆组件）
**Fact**：Guardian **不处理 Agent 记忆**——检查对象是工具调用结构化字段，无 RAG/记忆上下文输入。跨会话维度由 run_id 关联 threat_events（同一 run 的轨迹可查），无记忆投毒检测。
**Edge 证据**：init.sql threat_events.run_id 字段 + 无 memory 模块（与 Aigis cross_session/memory 模块对照——品类内覆盖差异）。

## 4.7 Policy Flow（治理闭环：规则 → 执行 → 反馈）
```
规则来源：
  静态：FORBIDDEN_CAPABILITIES / 9 regex 族 / approved+revoked registry（代码 + DB seed）
  动态：threat_rules 表（60s 热加载）——CAPABILITY_BLOCK/INJECTION_PATTERN/SEQUENCE_BLOCK
执行：check 0-6 顺序评估 → 决策
反馈：
  threat_events 表 → 运行时可查（/rules 端点 + invalidate-cache）
  DAST（ZAP 主动扫描，failOnError）→ CI 门禁
  治理文档：SECURITY.md（SAMM + Action SHA 固定 + 私有报告）约束开发过程
  canary 触发 → CANARY_TRIGGERED（探测信号进威胁事件流）
```
**Edge 证据**：init.sql threat_rules + app.py:948-1034（check 6）+ 1162-1194（rules 端点）+ .github/workflows/dast.yml + SECURITY.md。

## Flow 交叉校验（每条 L1 事实可回溯 Flow Edge）
| EK | Flow 边 | 可回溯 |
|---|---|---|
| EK-01 | Control | app.py:1194-1362 check 编排 |
| EK-02/03/04 | Authority | app.py:611-712 + init.sql registry |
| EK-05 | Data/Control | app.py:853-882 + 255-264 |
| EK-06 | Control/State | app.py:885-917 + agent_profiles |
| EK-07 | Data/Authority | app.py:918-947 |
| EK-08 | Policy | app.py:948-1034 + threat_rules |
| EK-09 | Evidence | app.py:768-851 + audit_log |
| EK-10 | Data/Control | app.py:1250-1280 + init.sql seed |
| EK-11 | Control | app.py:1199-1220 fail-safe 分支 |
| EK-12 | Policy | .github/workflows/ + SECURITY.md |
