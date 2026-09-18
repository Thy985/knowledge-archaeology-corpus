# 00 — Overview：Aigis（pyaigis-kr，确定性 Agent 防火墙）

## 一句话定位

Aigis 是**确定性、零依赖**的 AI Agent 运行时防火墙：每次工具调用前跑 L1-L7 检测管线（正则/语义/解码/CaMeL taint/沙箱/安全规约/FSM），产出可解释的 allow/deny 决策 + HMAC 签名防篡改审计日志，覆盖 MCP rug-pull、记忆投毒、间接注入、外泄通道等攻击面，并带 44→53 项合规策略模板（本 fork 聚焦韩国 PIPA/ISMS-P/금융위）。

## 为什么选它（Job Selection 摘要）

- 雷达 #10（09-11）建 A 级候选 `[cand]agent-firewall-runtime-defense-2026`，品类 09-01~09-12 一周内 10+ 项目涌现（Aigis/ClawKeeper/AgentGuard/Pipelock/OWASP Memory Guard/Guardian…）；雷达 #10/#11/#12 **连续三日**把"三选一实测"排为第一验证优先级。
- Aigis 是品类代表载体（信息密度最高：4-wall + L1-L7 + MCP 3-stage + 事件生命周期 + 44/53 合规模板 + 防篡改审计）。
- 与已考古 Corpus 连接：Agent 安全叙事与 dsh-pentest（未考古）、silver-shield（未考古）同线；与 agentevals（评测门禁）互补覆盖 Agent 工程两翼。

## 三个核心发现（Top Findings）

### F1 — "确定性判定"是硬路线：无 LLM、无启发式，全部规则可解释
L1 正则（165+ patterns / 25+ 类别 / 4 语言）+ L2 语义相似度（56 短语词典，仅补 L1 缺口防双重检测）+ L3 条件解码（仅编码指示时激活）→ 每步产出 MatchedRule（rule_id/score_delta/owasp_ref），决策链完全可回溯。基准自述 **93.5% detection（144/154）+ 0.0% FP（0/26）**，10 个 miss 明确标注为 alignment-frontier（sandbox escape 等）"not claimed as solved"——能力边界诚实到罕见。

### F2 — 纵深防御分层不是概念，是 7 层可执行管线
L1-L3 输入面 → L4 CaMeL capability-based access control（taint 追踪 × capability tokens → allow/deny）→ L5 Scan-Execute-Vaporize 原子沙箱 → L6 声明式安全规约（no_exfil/no_exec/pii_guard）→ L7 goal-conditioned FSM（AgentStateMachine 状态违规监控）。每层有独立代码模块（filters/safety/spec_lang），非 PPT 架构。

### F3 — 审计日志用 HMAC + hash chain 防篡改，且测试族直接验证"篡改即失败"
SignedAuditLog：entry frozen（dataclass frozen）+ canonical JSON + HMAC-SHA256 + prev_hash 链；测试族 test_tampered_action_fails_signature / test_tampered_risk_score_fails / test_tampered_outcome_fails / test_wrong_key_fails_verification——**"证据不可抵赖"被当作一等公民实现并测试**。

## 质量指标速览

- EK：16 条；KO：4 个；Candidates：4 个
- 测试证据：**1711 passed / 1 failed**（67s；唯一失败=test_i18n 的 LC_ALL 环境耦合——沙箱全局 LC_ALL=en_US 触发测试未 mock 的 POSIX 优先级，非产品缺陷，见 EK-16）
- 独立验证：ACCEPT（详见 06）

## 已知边界（诚实声明）

- **本快照是 pyaigis-kr fork（v1.1.4，韩国合规扩展）**；README/ARCHITECTURE 描述原版 Aigis（killertcell428/aigis，v1.1.0）能力——能力证据以本仓库实际代码为准，基准数字为原版自述。
- GitHub 仓库 **3 个月未更新**（pushed 2026-05-18）——品类活跃但本载体自身停滞；PyPI 通道 pyaigis-kr 存在（本次安装未验证最新版本号差异）。
- 检测基准（93.5%/0.0% FP）是**项目自述**（S7），未在本环境复跑 benchmark（需要完整 benchmark 数据）。
