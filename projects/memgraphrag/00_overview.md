# MemGraphRAG — Project Archaeology Overview

- **run_id**: ARCH-2026-09-06-001
- **project**: memgraphrag
- **repository**: https://github.com/xmudeeplit/memgraphrag.git
- **commit**: cd6fabda1ec31a302bb283089e17c087ab7713e8（main，"update readme"）
- **skill_version**: knowledge-archaeology v3.2
- **timestamp**: 2026-09-06（UTC+8）
- **mode**: initial

## 一句话定位

**MemGraphRAG 是一个记忆增强的 GraphRAG 框架**（KDD'26，XMU DeepLIT）：把语料组织成 Schema（本体三元组）/ Fact（关系三元组）/ Passage（原文块）三层记忆，经"多 Agent 组协同构建"（实际为多角色 LLM prompt 管道）生成记忆图，检索时以"embedding 相似度 + Personalized PageRank"融合排序。核心命题：用三层记忆 + 冲突感知构建 + 图增强检索提升多跳问答的可靠性与可追溯性。

## 为什么值得考古（Knowledge Value）

1. **Agent/AI Engineering 相关性（A 级）**：标题即 "Memory-based Multi-Agent System"——2026 RAG 从"静态检索"走向"agent 主动遍历"的收敛点；多角色 LLM 编排（NER/OpenIE/ontology/fact-check/curator/QA/IRCoT）是 Agent 管道的典型工程样本。
2. **Novelty**：三层记忆（Schema/Fact/Passage）双向链接 + 冲突感知构建（确定性候选生成 + LLM 判定分离 + 连通分量批量解析）+ 记忆派生图（type/entity/passage 三类节点）的实现细节。
3. **Benchmark 学习价值**：冲突检测的"结构候选 + LLM 判定 + 证据交叉匹配"模式、"检索 fallback（无 fact → dense）"模式、"content-filter fallback"模式均为可迁移工程模式。
4. **Design-Reality Gap 样例**：README 自称 "Multi-Agent System"，实现是**多角色 prompt 管道 + 多线程并发**（无 agentic 循环/无工具调用）——设计命名 vs 实现现实的典型考古对象。

## 认知核心（一句话）

**"多 Agent"未必是 agentic 循环——多角色 LLM 管道 + 确定性结构逻辑 + 显式 fallback 也能构成可用的"agent 化"系统**：MemGraphRAG 把不确定的认知（本体归纳/冲突判定/QA 推理）交给 LLM 角色，把确定性的结构（候选生成/图构建/频率归一化/PPR）交给代码，二者分离。

## 关键数字（均可回溯）

| 指标 | 值 | 来源 |
|------|-----|------|
| 规模 | 124 文件 / 46 py / 7,330 行 Python | find+wc |
| 版本 | 无 tag（main HEAD cd6fabd，"update readme"） | git |
| 许可证 | MIT（Copyright 2026 DeepLIT Group, Xiamen University） | LICENSE |
| 核心文件 | MemGraphRAG.py 2,339 行（管线编排）/ Memory.py 503 行 | wc |
| 三层记忆 | Schema 1:N Fact N:M Passage（双向链接） | Memory.py docstring |
| 检索 | embedding 相似度 + PPR（damping=0.5, prpack） | run_ppr |
| 冲突处理 | 确定性候选 + LLM 判定（temperature=0）+ 连通分量解析 | detect/resolve |
| 测试 | **无测试目录**（0 测试文件） | find tests |
| 依赖 | openai/litellm/vllm/gritlm/torch/transformers/networkx/python_igraph | requirements.txt |

## 产物清单

- `01_project-layer/` — 项目地图（架构/模块/数据结构/状态/配置/治理/依赖）
- `02_engineering-knowledge/` — EK Graph（30 条 EK + 6 类边）
- `03_knowledge-layer/` — Generalized KO（8 个 Core KO + 聚合规则 R1-R4）
- `04_flow-atlas/` — 七类流（Control/State/Data/Evidence/Authority/Memory/Policy）
- `05_candidates/` — 未确认假说（含"Multi-Agent"命名差距、eval 安全边界等）
- `06_validation/` — Validation 报告 + 反例 + Reconciliation + 质量指标
- `run_metadata.yaml` — run 元数据

## 免责声明

本包所有事实均可回溯到 cd6fabd 快照实际内容。验证阶段"真实代码行为"来自直接执行 repo 中模块的实测结果（ThreeLayerMemory 纯 Python 构建/双向链接/序列化往返）；LLM 依赖管线（OpenIE/冲突判定/QA）因环境无 openai/embedding 模型未实跑，仅静态精读，诚实标注。
