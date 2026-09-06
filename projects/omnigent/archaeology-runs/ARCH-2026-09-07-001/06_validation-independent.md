# Independent Validation Report — ARCH-2026-09-07-001 · omnigent

> Auditor 流程：不把 Archaeology Result 当事实来源；独立重读仓库（重点：runner/policy.py 全文段、policies/builtins/、inner/datamodel.py CredentialProxy、onboarding/sandboxes/kubernetes.py、sandbox/、server/routes/_sessions/、telemetry.py）建立 Independent Findings 后，再与考古包比对。
> 本报告独立于 package/ 06_validation.md 生成；**未修改任何考古产物**。

## 一、判定统计

| 判定 | 数量 | 代表 |
|------|------|------|
| CONFIRMED | 26 | EK-01~06,08~28,30 主体；KO-01~07；CM-01~04；M-01~04 |
| PARTIALLY_CONFIRMED | 3 | EK-29（文档 proposed vs 代码已有部分改进）；KO-05（SQLite 超时未自愈）；04.5 事件 URL 字面 |
| DOWNGRADED | 0 | — |
| OVER_GENERALIZED | 0 | — |
| MISSING | 1 | CredentialProxy 机制（swap-on-access + placeholder + cross-host leak guard） |
| CONTRADICTED | 0 | — |
| NEEDS_HUMAN_REVIEW | 1 | C-07（tool_dispatch 上送 vs policy gate 顺序）；原 #1（git 凭据流）已由 Auditor 解决 |

## 二、Auditor 独立发现（Independent Findings，先于比对）

1. **CredentialProxy（inner/datamodel.py:400-430）**：每个 `credential_proxy` YAML 类型（https_bearer/https_basic/git_https/gh_basic）被归一化为 CredentialProxyEntry；**默认 swap-on-access**——沙箱持有"nothing credential-shaped"，egress MITM proxy 在出站时注入 `Authorization`；**opt-in env injection**——`gh` 等客户端本地看不到凭据就短路"authentication required"，此时注入合成 `oa_cred_*` 占位符让客户端发出请求，proxy 在出口换真 secret，且**占位符重放到其它 host → HTTP 403（cross-host leak guard）**。
2. **kubernetes.py:492-497**：沙箱内 `configure_clone_credentials(server_url, host_id)` 是**配置代理绑定**（server_url + host_id 定位），非把真凭据写入沙箱——git 凭据流与"用户凭据不进沙箱"兼容（佐证 KO-06，解决原 NEEDS_HUMAN_REVIEW #1）。
3. **runner/policy.py:270-284**：组合语义与 server 侧 mirror——"DENY short-circuits immediately — no later policy can override a denial; ASK is recorded but evaluation continues; ALLOW continues; after all policies: return the first recorded ASK (if any), else ALLOW"。**DENY 不可覆盖是硬规则，无 override 旁路**。
4. **绕过路径自知（policies/builtins/）**：`_shell.py:9`（首 token 会被平凡绕过→策略边界）、`cost.py:38`（防止 unpriced spend 静默绕过 cap）、`safety.py:37`（工具绕过 sys_os_* 直接执行）——策略系统明确知道自己可被绕过并针对性设计（负向证据，佐证 KO-03 边界）。
5. **telemetry.py:381**：代码已有 "continues an incoming traceparent across each HTTP boundary"——OBSERVABILITY.md 的 proposed 状态与代码部分改进**并存**（时间差未在考古包声明）。
6. **server/routes/_sessions/**：approval/elicitation 路由族存在（_antigravity_elicitation.py / _codex_elicitation.py / orchestration.py），但 04.5 字面 URL `/v1/sessions/{id}/events` 未在路由文件直接核实——路径存在、字面 URL 待确认。

## 三、重点攻击结果

| 攻击面 | 结果 |
|--------|------|
| 单案例→Pattern | KO-01~07 均标 Cross-project validation pending，无违规升层。✅ |
| Pattern→L4 | CM-01~04 均有一句话稳定关系 + 强单项目证据；无"文字抽象但解释范围未扩大"。✅ |
| 项目经验→通用 Principle | C-04/C-06 留在 Candidates（Hypothesis/Tentative）。✅ |
| ADR→实现事实 | EK-29 文档真实但需补"代码已部分实现"（见 PARTIALLY_CONFIRMED）。⚠️ |
| Flow Edge 真实性 | 04.1~04.7 核心 Edge（evaluate_policy=True / DENY 文本 / launch token / owner.pid / 86400s）全部 grep 实证；仅事件 URL 字面待核实。✅/⚠️ |
| bypass/override/exception/alternate | DENY 无 override（硬规则）；绕过路径自知（_shell/cost/safety）；无隐藏 admin path 发现。✅ |
| Epistemic 混淆 | 无 Hypothesis 冒充 Fact；S2/S3 分级正确。✅ |

## 四、必须回答的问题

### 原 Archaeology 最重要的 3 个成功
1. **策略执行点跟随控制权 + 双评估**（EK-03/04）：架构决策有自述证据（runner/policy.py），是 meta-harness 控制面最可迁移的洞察。
2. **审批超时语义修复史**（EK-08）：120s→86400s 把"静默拒绝"认定为 bug——人机闸门默认信任方向的绝佳案例（CM-02 成立）。
3. **身份持久 vs 资源易逝 + 凭据隔离**（EK-18/19）：managed_hosts 的 host_id 绑定设计 + 沙箱凭据隔离，代执行环境安全设计的骨架完整。

### 最重要的 3 个错误
1. **MISSING CredentialProxy 机制**：考古包只把 git 凭据列为 NEEDS_HUMAN_REVIEW，未找到机制主体（swap-on-access/placeholder/403 leak guard）；Auditor 独立发现后补 EK-31（Reconciliation 阶段合入 corpus）。
2. **04.5 事件 URL 字面未经核实**：`/v1/sessions/{id}/events` 写得太具体，实际路由族在 server/routes/_sessions/；应改为"server/routes/_sessions/ 事件路由（elicitation/approval）"。
3. **EK-29 时差未声明**：文档 Status: Proposed 与 telemetry.py 已有 traceparent 延续代码并存，考古包未标注"文档状态 vs 代码现状"时间差。

### 是否存在关键遗漏？
是——CredentialProxy（见错误 1）。它强化 KO-06 的凭据隔离（占位符 + 403 泄漏守卫是"凭据永不进沙箱"的工程化实现），并提供一个跨 host 泄漏守卫的 regression 语义。

### 是否存在错误升维？
否。所有 L3+ 均带 pending 标注或强证据；无"漂亮结论"升层。

### 是否存在事实错误？
无直接事实错误。一处字面 URL 过度具体（04.5），一处状态时差未标注（EK-29）。

### 是否存在 Flow 错误？
04.5 Authority Flow 的 approval 事件 URL 字面需修正；其余 Edge（evaluate_policy=True 回环、DENY 文本返回、launch token、owner.pid 收割、86400s 等待）全部实证可回溯。04.7 Policy Flow 治理闭环（RUNNER_MCP→PR 门槛→PolicySpec→runner/server 双面→CHANGELOG 反馈）成立。

### 是否发现新的 Benchmark / Regression Case？
- **R-01（回归语义）**：CredentialProxy cross-host leak guard——占位符 `oa_cred_*` 重放到非绑定 host 必须 403；反向（正确 host 注入真实凭据）必须成功。候选 fixture：CredentialProxyEntry 解析 + egress 拦截。
- **R-02（回归语义）**：policy gate DENY 不可覆盖——构造"前策略 DENY + 后策略 ALLOW"，结果必须 DENY（且后策略不得改变裁决）。
- **R-03（负向 case）**：`_shell.py` 首 token 平凡绕过边界——策略文档承认的边界应被测试显式断言（哪些输入"应该"绕过并明确注释），防止未来误当 bug 修掉。
- **B-01（benchmark）**：harness 能力声明（HarnessCapabilities）解析——5 集成模式 × 4 elicitation 通道 × capabilities 表的一致性检查（防回归到隐式分支）。

## 五、Reconciliation 建议（阶段 6 合入 corpus，不改考古包）

1. 补 EK-31（CredentialProxy：swap-on-access + placeholder + cross-host leak guard；证据 inner/datamodel.py:400-430 + kubernetes.py:492-497）。
2. 04.5 事件 URL 修正为 "server/routes/_sessions/ 事件路由（elicitation/approval）"。
3. EK-29 补注：telemetry.py 已有部分 traceparent 延续实现（telemetry.py:381），OBSERVABILITY 文档为 proposed 审计基线。
4. KO-06 补强证据：CredentialProxy 占位符 + 403 守卫（EK-31 链接）。
5. NEEDS_HUMAN_REVIEW #1（git 凭据流）关闭：确认 configure_clone_credentials 为代理绑定配置，非真凭据入沙箱。
