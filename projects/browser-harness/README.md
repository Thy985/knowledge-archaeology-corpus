# browser-harness — Knowledge Archaeology

**一句话定位**：极简自愈浏览器 harness（browser-use 出品，MIT，15.7k+ stars，v0.1.13）——LLM 经一根可编辑 CDP WebSocket 直连真实 Chrome，agent 在执行中自写缺失 helper（agent-workspace/agent_helpers.py 运行时注入），"harness 每次运行自我改进"。无框架/无 recipes/无 rails。

## 产物清单
| 目录/文件 | 内容 |
|---|---|
| `metadata.yaml` | 考古元数据（run_id / commit / skill_version） |
| `00_overview.md` | 概览：一句话定位 + 核心认知预览 |
| `01-project-layer/` | 项目地图（架构/模块/数据结构/生命周期/配置/治理） |
| `02-engineering-knowledge/` | EK Graph（47 条 + 六类边） |
| `03-knowledge-layer/` | Generalized KO（8 个，R1-R4 聚合规则） |
| `04-flow-atlas/` | 七类流（Control/State/Data/Evidence/Authority/Memory/Policy） |
| `05-candidates/` | 未验证假说（7 个，含 SKILL.md 身份冲突 NEEDS_HUMAN_REVIEW） |
| `06-validation/` | 验证报告 + 独立审计 + Reconciliation |
| `archaeology-runs/ARCH-2026-10-04-001/` | 本次 Run 原始产物（manifest/snapshot/metadata/审计） |

## 考古日期与版本
- 考古日期：2026-10-04（run ARCH-2026-10-04-001，mode: initial）
- 目标版本：browser-harness 0.1.13 @ `afbcc381`（2026-09-07，Merge PR #757 video-export-nested-output）

## 知识贡献（summary）
- 三层配比：100+ Facts / 47 EK（L1×12 + L2×35，平均出边 2.3，游离 0）/ 8 KO（R1×2 R2×2 R3×1 R4×3）/ 7 类 Flow / 7 Candidates。
- 关键认知：**KO-01 进程身份端到端验证**（ping 协议 {pong:true} 严格 dict + identify type-is-int + start-time 指纹 + generation 指纹，SIGTERM 前双重确认）；**KO-03 被拒不重连纪律**（permission 拒绝=状态非错误，防"一次批准变无尽弹窗"）；**KO-04 受控中间层 + 运行时能力扩展**（daemon 持会话权威、agent 在工作区自写 helper）；**KO-05 cloud 计费权威**（shutdown 失败保持 daemon 存活重试清理，防孤儿化计费浏览器）；**KO-06 四层输出面统一脱敏**（日志拓扑/录制 URL scrub/遥测 FORBIDDEN_KEYS/video SENSITIVE）。
- 独立审计（盲重建）：CONFIRMED 8 / PARTIALLY_CONFIRMED 2 / DOWNGRADED 2 / OVER_GENERALIZED 0 / MISSING 5 / CONTRADICTED 0 / NEEDS_HUMAN_REVIEW 1（SKILL.md `name: browser-harness` vs AGENTS.md identity `browser-use` 冲突）。Reconciliation 合并 8 项（E-01 domain-skills 首标签 / E-02 超时三层预算 / E-03 P-02 范围限定 / M-01..05 → EK-46 dialog 状态机、EK-47 debug-clicks、EK-34/35/04 修订）。
- Benchmark case：BK-01 守护进程身份验证（与 deepseek-harness daemon 管理可对照，gold record 候选）。

## 关联
- 同厂对照：`browser-use`（09-22 考古，LLM 决策层）vs 本仓库（CDP 连接层）——互补。
- 连接：E2E-CLI / campus_order(OpenClaw) / agent-attention / Computer Use（G5 S5 领域）；EP-002 权限边界（自愈写代码=高风险权限需边界）。
