# Evolver — Knowledge Archaeology Index

**一句话定位**：GEP（Genome Evolution Protocol）驱动的 AI Agent 自进化引擎（EvoMap，v1.94.0，9k stars，arXiv 2604.15097）——扫描运行日志→匹配基因/胶囊→生成协议约束进化 prompt→solidify 验证→学习回填，配套 ATP 经济层与 EvoMap Hub 进化网络。

**考古日期**：2026-09-06 ｜ **run_id**：ARCH-2026-09-06-002 ｜ **commit**：31b0691（main） ｜ **skill**：knowledge-archaeology v3.2 ｜ **mode**：initial

## 产物清单

| 层 | 文件 | 内容 |
|----|------|------|
| Overview | 00_overview.md | 定位/价值/认知核心/免责声明 |
| Project Layer | 01_project-layer.md | 六面架构 + 混淆状态图 + 数据结构 + 权限/治理 |
| Engineering Knowledge | 02_engineering-knowledge.md | 30 EK（EK Graph，6 类边，100% links） |
| Knowledge Layer | 03_knowledge-layer.md | 7 KO（R1-R4 聚合）+ 4 CM + 4 M |
| Flow Atlas | 04_flow-atlas.md | 七类流（Control/State/Data/Evidence/Authority/Memory/Policy） |
| Candidates | 05_candidates.md | 8 候选（2 个 NEEDS_HUMAN_REVIEW） |
| Validation | 06_validation.md | 反例 12 次 + Epistemic 清单 + 质量指标 |
| Run 快照 | archaeology-runs/ARCH-2026-09-06-002/ | run_metadata.yaml + 全部层副本 |

## 关键发现（摘要）

1. **自进化闭环**：信号→选基因（不 improvisation）→协议 prompt→solidify 验证→soft/hard 失败分类→stash 回滚→学习回填（KO-01）
2. **安全纵深**：node-only 白名单（npm/npx 因 GHSA 移除）+ flag 阻止 + 元字符拒绝 + 路径遏制 + A2A 资产隔离 + 源码级回归测试（KO-02）
3. **自我更新可恢复**：备份/日志/原子重命名/金丝雀/崩溃恢复 fail-closed（KO-03）
4. **混淆治理现实**：57/202 src 文件混淆（evolve 全模块 + gep 半数）+ GPL-3.0 声称 + source-available 转向公告——开源承诺 vs 可审计性（KO-07）

## 关键争议（需人类裁决）

- C-02：README 白名单 node/npm/npx vs 实现 node-only（doc-impl 矛盾方向：文档比实现宽松）
- C-03：README "prompt generator, not code patcher" vs solidify 修改 src/**（自进化引擎"到底改不改自己的码"核心语义，混淆实现不可读）

## 盲区声明

src/evolve.js 与 evolve/pipeline/*（8 文件）及 gep 44 文件被混淆，内部逻辑以外部观察（CLI 调用点/README/测试契约）为据；抽样测试 60 pass / 2 fail（fail 因 @evomap/gep-sdk 未安装，非回归）。
