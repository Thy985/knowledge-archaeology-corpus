# MXC（Microsoft eXecution Container）— Knowledge Archaeology

**定位**: 微软开源（MIT）OS 级策略驱动 agent 遏制沙箱引擎（v1.0.1，Rust 246K 行）——"agent 不能成为自身的安全权威"，统一 containment model + 版本化 JSON 契约 + 9 后端 + Rust/.NET/Node SDK。

**考古日期/版本**: ARCH-2026-10-24-001 · skill v3.2 · HEAD `ffecec0`（2026-10-22）
**status**: COMPLETED（initial）

## 认知主线（见 00_overview / 03_knowledge-layer）
1. fail-closed 是链条不是口号（closed JSON → 默认 deny → FS fail-closed → 后端拒绝 → telemetry fail-closed）
2. 契约/运行时/授权三轴分离（"哪个字段存在"≠"哪个后端运行"≠"主机能强制什么"）
3. 能力边界诚实（IsolationSession 显式全 allow / WSLc 双态 / Seatbelt 逃生门 / proxy 明示限制）
4. Learning→Enforcement 渐进（deny-and-record 先观察，permissive 显式弱化，plm 生成最小权限候选）
5. 治理代码化（AGENTS.md/copilot-instructions 把"行为覆盖+不发明证据"固化为评审标准）

## 产物清单
| 目录 | 内容 |
|---|---|
| 01_project-layer/ | 项目地图 |
| 02_engineering-knowledge/ | EK Graph（M01~M20，Reconciliation 后 20 条） |
| 03_knowledge-layer/ | KO-M01~M06（R1×1 R2×2 R3×1 R4×2；L3×3 L4×3 全 pending 标记） |
| 04_flow-atlas/ | 七类流 |
| 05_candidates/ | C-M01~M07（Hypothesis / Cross-project pending） |
| 06_validation/ | Validation + Independent Validation（11 CONFIRMED/1 PARTIALLY/0 CONTRADICTED）+ Reconciliation（R-1~R-5） |
| archaeology-runs/ARCH-2026-10-24-001/ | run_metadata / job-manifest / snapshot |

## 跨项目连接
- **strands-box**（10-23，双对照组）：Box 胜策略语言+时间性+凭证网关；MXC 胜平台级多后端+组织策略+三模式
- **dogwood**（09-08）：策略语言本体；**deepseek-harness**（validated）：sandbox/approval 锚点；**openclaw**（09-17）：MXC 生态首批接入
- **SIP**: 无（本轮未产生 Skill Improvement Proposal）
