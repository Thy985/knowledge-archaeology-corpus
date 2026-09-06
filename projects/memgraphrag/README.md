# MemGraphRAG — 项目考古索引

**一句话定位**：记忆增强的 GraphRAG 框架（KDD'26，XMU DeepLIT）——三层记忆（Schema/Fact/Passage）+ 冲突感知构建 + 图增强检索（embedding 相似度 + Personalized PageRank），标题宣称 "Multi-Agent System"，实现为多角色 LLM prompt 管道。

**产物清单**：
- `00_overview.md` — 概览与认知核心
- `01_project-layer.md` + `01_project-layer/` — 项目地图
- `02_engineering-knowledge.md` + `02_engineering-knowledge/` — EK Graph（33 条）
- `03_knowledge-layer.md` + `03_knowledge-layer/` — Generalized KO（8 KO / 4 CM / 4 M）
- `04_flow-atlas.md` + `04_flow-atlas/` — 七类流
- `05_candidates.md` + `05_candidates/` — 9 候选（含 IRCoT 死代码、eval 边界）
- `06_validation.md` + `06_validation/` — Validation + 反例
- `07-independent-validation.md` — 独立 Auditor 报告（盲重建）
- `archaeology-runs/ARCH-2026-09-06-001/` — 本次 run 完整快照（含 job-manifest + run_metadata）

**考古日期**：2026-09-06 ｜ **run_id**：ARCH-2026-09-06-001 ｜ **commit**：cd6fabd（main） ｜ **skill**：v3.2

**关键发现速览**：①三层记忆数据结构实测验证 ②冲突三阶段分离（确定性候选→LLM 判定→分量解析）③"Multi-Agent"名实差距（无 agentic 循环，IRCoT 死代码）④检索信号融合 + 显式 dense fallback ⑤eval() 安全边界。
