# 04 Flow Atlas — 七类流（OWASP AST10）

> 文档型仓库：Control/State/Data/Evidence/Policy 直接锚定 docs/diagrams 与 check.py 等真实符号；Memory/Authority 锚定风险页与 Universal Format 字段。每条流回答一个问题，Edge 可回溯 symbol。

## 1. Control Flow — skill 执行请求如何被裁决
- **question**: 谁决定一个 skill 动作是否执行？
- **chain**:
  - LLM（产生动作意图）
  - Skill（编排工具调用）→ `ast09.md:Execution Receipts`（action_type）
  - Tool Request（形成具体请求）
  - **Policy Enforcement Layer**（裁决）→ `docs/diagrams/execution-boundary-flow.md`
  - ALLOW → Execution；DENY → Block + Trace ID
  - 决策返回调用层（proceed / retry / escalate / abort）→ `execution-boundary-flow.md:decision returned`
- **gates**: 副作用前确定性判定（"evaluate the request before side effects occur"）；`policy_version` 参与决策（ast09 admission receipt）
- **states**: PENDING → ALLOWED / DENIED / ESCALATED → EXECUTED / BLOCKED
- **evidence**: [execution-boundary-flow.md][ast09.md]

## 2. State Flow — skill 安装/运行/更新状态如何演变
- **question**: skill 从安装到弃用的状态机？
- **chain**:
  - Registry（可投毒态）→ `ast02.md`（发布门槛一周龄账户）
  - Install（metadata 解析态，加载即执行风险）→ `ast04.md`（"deserialized during the skill-loading lifecycle"）
  - Execute（host 上下文态，全权限）→ `ast03.md`/`ast06.md`
  - Update（漂移态：缺补丁 or 伪补丁）→ `ast07.md`
  - Port（跨平台元数据丢失态）→ `ast10.md`
- **states**: REGISTRY → INSTALLED → EXECUTING → UPDATED/DRIFTED → PORTED/STRIPPED → GOVERNED
- **gates**: 版本 pinning（content_hash）；risk_tier 分级（L0-L3，Universal Format）
- **evidence**: [ast02/ast04/ast06/ast07/ast10.md][universal-skill-format.md]

## 3. Data Flow — 安全元数据如何产生、传递、丢失
- **question**: 权限/签名/扫描元数据从发布到执行如何流转？
- **chain**:
  - author 声明（identity did:web + signing_key ed25519）
  - manifest 编码（permissions.files/network.allow/shell/risk_tier/scan_status）→ `universal-skill-format.md`
  - registry 校验（Merkle root 签名）→ `ast01.md`
  - 移植（格式转换 → 元数据剥离）→ `ast10.md`（metadata-loss-simulator）
  - 执行（policy layer 消费权限声明）
- **data_forms**: YAML manifest → canonical hash（JCS→sha256）→ receipt 字段
- **conservation_points**: attempt_id = 内容哈希（篡改即失效，check.py 可重算）→ `examples/ast09-execution-receipts/check.py`
- **evidence**: [universal-skill-format.md][ast09.md][check.py]

## 4. Evidence Flow — 安全声明如何变可证明
- **question**: 如何证明一个 skill 动作"确实被阻断"或"确实发生"？
- **chain**:
  - admission receipt（执行前 7 字段：attempt_id/agent_id/action_type/scope/policy_version/decision/timestamp_ms）
  - outcome receipt（执行后：attempt_id/action_ref/terminal_state）
  - 离线验证（check.py：7 字段存在 + decision=DENY + attempt_id 重算；denied-before-dispatch 属性）→ `check.py`（exit 0/1/2）
- **gates**: 内容派生标识（"altering any field in the preimage makes the stored attempt_id fail to recompute"）
- **evidence**: [ast09.md:Execution Receipts][check.py:docstring][deny-admission-receipt.json][deny-admission-receipt.tampered.json]

## 5. Authority Flow — 谁有执行权
- **question**: skill 的执行权如何授予、作用域与约束？
- **chain**:
  - agent 权限（host 全权限）→ `ast03.md`（OpenClaw host-mode 证据）
  - per-skill scope（缺失）→ 声明式 manifest（Universal Format permissions）
  - 意图级授权（缺失，LPCI 漏洞）→ `ast03.md` LPCI（工具调用层检查 ≠ 意图层）
  - deny_write 保护（身份文件显式授予）→ `universal-skill-format.md`
- **bypass/override**: 权限提升注入（LPCI）可绕过 manifest（encoded/delayed/conditional payload）；host-mode 无沙箱 = 默认全权
- **evidence**: [ast03.md][universal-skill-format.md]

## 6. Memory Flow — 记忆/身份如何沉淀与被污染
- **question**: agent 的记忆文件如何成为持久后门？
- **chain**:
  - MEMORY.md/SOUL.md（身份/记忆载体）→ `ast01.md`（Soul Persistence）
  - 恶意写入（backdoor 指令持久）→ `ast01.md` Memory Poisoning
  - 身份克隆（读取复制行为状态）→ `ast01.md` Identity Cloning
  - 防护（deny_write 默认拒绝）→ `universal-skill-format.md`
  - 外部窃取（Vidar 变体窃 openclaw.json/soul.md/memory.md）→ `index.md` Incident Feb 2026
- **states**: TRUSTED → POISONED → PERSISTED → CLONED / PROTECTED
- **evidence**: [ast01.md][index.md][universal-skill-format.md]

## 7. Policy Flow — 系统如何改变自己"下一步"的规则（治理闭环）
- **question**: skill 安全策略如何被制定、固化、执行并反哺未来决策？
- **chain**:
  - **Decision**: OWASP 威胁分类（AST01-10 立项）→ `proposal.md`
  - **Approval**: OWASP Incubator 治理 + leaders 评审 → `leaders.md`/`info.md`
  - **Policy**: 固化为检查清单 + Universal Format 规范（deny_write/risk_tier/scan_status）→ `checklist.md`/`universal-skill-format.md`
  - **Enforcement**: 注册表扫描（ClawHub VirusTotal/行为扫描）+ policy enforcement layer（ALLOW/DENY）→ `skill-scanner-integration.md`/`execution-boundary-flow.md`
  - **Future Decision**: incident-response 剧本 + 新 AST 条目提交表单 + 企业映射（AI TIPS/MAESTRO）→ `incident-response.md`/`community-contribution.md`/`mappings.md`
- **gates**: v1 公开评审 Google Doc；OWASP Flagship 晋升（Q4 2026）→ `index.md:Project Status`
- **evidence**: [proposal.md][leaders.md][checklist.md][universal-skill-format.md][mappings.md]

## 交叉校验（Flow→KO）
- Control(1)+Authority(5)+Policy(7) 重复"Decision→Gate→Enforcement→Record"结构 → 支撑 KO-03/KO-04
- Memory(6)+Data(3) 的 deny_write/attempt_id 守恒 → 支撑 KO-05/KO-07
- Evidence(4) 的收据双份 + 独立重算 → 支撑 KO-06 收据段
- 所有 Edge 均可回溯至本仓库符号（文件/字段/exit code），无概念箭头
