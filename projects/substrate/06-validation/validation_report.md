# Validation & Evidence — Agent Substrate（06 主考古 Validation）

> 六 Auditor 视角 + Blind Reconstruction 前置（盲重建在阶段⑤独立进行，此处记录主考古自检与待验证清单）。

## Truth Auditor（真值）
- 已核：proto 枚举/记录字段（七态/三级 fidelity/durable_snapshot 双态）为代码事实；egresspolicy Compile/EvaluateRequest 逻辑与文档一致
- 待核（记入 IVR 攻击面）：security.md "Met" 声明 vs 代码（如 Dual-Party Snapshot Authorization 的 GCS token exchange 实现）、request-parking 实现、atunnel 443/8443 实际监听与 egress 路径

## Coverage Auditor（覆盖）
- 已覆盖：控制面 API 模型（proto 全清单）、atelet 生命周期（proto 注释契约）、ateworker checkpoint 幂等、microVM 快照（run.go/checkpoint.go）、gVisor 后端（文件级）、egresspolicy 编译评估、身份（actoridjwt/credbundle）、授权（authorization.md+ateapiauth）、网络（atunnel/tunnel.go+atenet 目录）、请求停车（request-parking.md）、资产链（sandbox_asset_cache）、治理（AGENTS.md）、安全状态体系（security/threat-model）
- 未精读（显式缺口）：cmd/ateapi 控制面实现体（scheduler/workflow engine/OpenFGA 集成）、internal/atunnel/egress 全实现（4904 行仅采样 tunnel.go）、localjwtauthority 主实现（621 行仅见测试文件）、objectstore-plugin、podcertcontroller、atecontroller、docs/authentication.md、e2e 测试全文——记入 IVR 遗漏

## Flow Auditor（流）
- 七类流已构建（04_flow_atlas），关键 Edge 均有锚；3 处 [pend] 需代码验证（C1 停车实现/S2 状态转换实现/D1 PromoteSnapshot 实现）
- 无 KO-Flow 矛盾

## Abstraction Auditor（升维）
- KO-S02/KO-S06 标 L4 pending（单项目/双项目证据），未冒充 Principle
- KO-S05 跨项目假说 C-D05 已标 Cross-project validation pending

## Counterexample Hunter（反例）
- KO-S04（停车）反例：立即 503 方案（parking 文档明示代价）；lot 满即 shed（背压而非无界排队）
- KO-S07（解耦）反例：普通 Pod 生命周期绑物理 Pod（对照）
- KO-S01（三件套）反例：dapr-agents durable execution（步骤日志 vs 整机快照——同一问题的不同解法，证"三件套"非唯一解）
- KO-S03（代行可区分）反例：单主体令牌方案（代行与自身同 sub——无法区分，即本项目避免的）
- 0 反例 KO：无（每条 L3+ 均有 ≥1 定向反例）

## Epistemic Auditor（认知状态）
- Fact/Observation/Hypothesis/Pattern/Model 标注逐条检查：EK 20 条 2 条含 Hypothesis（S19 未实现面）；KO 7 条 3 条 L4 pending；Candidates 7 条全 Hypothesis
- 无"假设冒充事实"；无"单项目经验冒充普适原理"

## Quality Metrics（本 run）
- Facts/Evidence: 采证锚 >40（proto/代码/文档/测试）
- EK: 20 条（目标 40-60 按规模缩放——仓库 42.7 万行但考古聚焦核心机制，保留宽底座原则；**声明：非全量**）
- Patterns/L3+: KO 7 条（2 条 L4 pending）
- EK Graph: 平均出边 2.25（≥1）；游离 EK 0/20（<20%）；聚合覆盖率 100%
- Flow: 7 类流 21 条关键 Edge（3 条 [pend]）
- Candidates: 7（含跨项目假说 C-S05）

## 待 IVR 判定清单（阶段⑤攻击面）
1. security.md "Met" 各项的代码实锤（特别是 Dual-Party Snapshot Authorization / No Access by Default 的 DNS 过滤）
2. WorkerAssignment 反规范化热路径是否真被 atenet 读取（proto 注释 vs 调用）
3. checkpoint 幂等测试是否覆盖超时重试场景
4. atunnel 8443/443 双监听与 `--upstream-connect=false` bypass 路径
5. request-parking 的 parking lot 实现存在性
6. egresspolicy 的 SNIRules/EvaluateRequest 是否与 PEP 实际接线（atunnel/egresspolicy.go）
