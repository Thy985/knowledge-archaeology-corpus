# Pipelock — Knowledge Archaeology（ARCH-2026-10-16-001）

开源 AI agent 防火墙（Verifiable Egress Control）：网络/工具代理中介 AI-agent HTTP/WS/MCP/A2A 流量，扫描 secret exfiltration / prompt injection / SSRF / tool poisoning，并对每次边界决策产出 **mediator-signed action receipts**（外部可离线验证证据链）。核心命题："agent 不能成为自身安全权威"。

| 产物 | 说明 |
|---|---|
| 00_overview.md | 三条主线 + 质量快照 |
| 01_project-layer/ | 项目地图（架构/模块/生命周期/测试/治理） |
| 02_engineering-knowledge/ | EK Graph 35 条（六类边） |
| 03_knowledge-layer/ | 8 个 Core KO（R1-R4 聚合规则） |
| 04_flow-atlas/ | 七类流 28 Edge |
| 05_candidates/ | 8 个未验证假说 |
| 06_validation/ | Validation + 独立盲重建 + Reconciliation |
| archaeology-runs/ARCH-2026-10-16-001/ | run_metadata + snapshot + job_manifest |

考古日期：2026-10-16 · commit `8ade100` · v3.7.0 · 独立验证：CONFIRMED 9 / PARTIALLY 1 / MISSING 3（已合并修正）/ NEEDS_HUMAN_REVIEW 2
