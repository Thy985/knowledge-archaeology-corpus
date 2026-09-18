# Guardian 项目考古索引

- **定位**：确定性 Agent 安全 Sidecar（legionforge-guardian v0.1.1）——在任意 Agent 框架执行工具调用前跑 7 项确定性检查（Task token ACL / registry / capability / destructive / sequence / hash / adaptive），无 LLM；PostgreSQL SHA-256 hash chain 审计 + canary 蜜罐工具。
- **考古日期**：2026-09-14（ARCH-2026-09-14-001，initial）
- **commit**：`75c1aaba928d8b1a9b32266081487e4863215f9c`（main）
- **skill 版本**：knowledge-archaeology v3.2
- **产出清单**：

| 产物 | 文件 |
|---|---|
| Overview（定位 + 3 Top Findings + 质量指标） | 00_overview.md |
| Project Layer（项目地图） | 01-project-layer/ |
| Engineering Knowledge（14 条 EK Graph） | 02-engineering-knowledge/ |
| Knowledge Layer（4 KO，R1-R4 聚合） | 03-knowledge-layer/ |
| Flow Atlas（七类流，可回溯） | 04-flow-atlas/ |
| Candidates（4 个未验证假说） | 05-candidates/ |
| Validation & Evidence（质量指标 + 反例） | 06-validation/ |
| Run 记录（run_metadata / snapshot / independent validation） | archaeology-runs/ARCH-2026-09-14-001/ |

- **核心结论**：①确定性判定把注入面从文本空间移到结构空间（与 Aigis 双项目互证）；②SHA-256 hash chain 审计与 Aigis HMAC 链同构（防篡改审计模式第二数据点）；③安全热路径 fail-closed / 审计 fail-open 的边界双项目独立出现；④canary 蜜罐为独特机制（单项目，pending）。
- **基线说明**：跨项目对照引用 Aigis（ARCH-2026-09-13-001，分支 `archaeology/aigis-ARCH-2026-09-13-001` commit 1ac73b2，**该 PR 尚未合并入 main**——引用的 Aigis 证据以本地考古包为准）。
