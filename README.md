# knowledge-archaeology-corpus

**已完成项目考古结果的存档库（Corpus）**。保存 knowledge-archaeology skill 对真实项目执行考古后产出的全部知识资产——包括原始提取、独立审计、失败分析、版本演进基线。

与 [knowledge-archaeology-skill](https://github.com/Thy985/knowledge-archaeology-skill) 的分工：
- **skill 仓库**：方法论本身（Multi-Agent Knowledge Archaeology System）
- **corpus 仓库**：用该方法论对具体项目做考古后，沉淀下来的**已完成考古结果**

## 当前内容：Codex 考古

对 OpenAI Codex（本地只读克隆）的多轮考古记录：

```
├── 00-skill-architecture.md           # skill 架构参考（v1 时点）
├── 01-repository-map.md               # 仓库地图
├── 02-evidence-packs/                 # Wave1 证据包
├── 04-mining-graphs/                  # Wave2 挖掘图
├── 05-knowledge-objects/              # Wave3 知识对象
├── 06-validation/                     # Wave4 验证 + 最终包
├── validation/                        # 独立对抗审计（10 文件）
└── benchmarks/codex/
    ├── failure-analysis.md            # v1 失败分析（Status: FAILED QUALITY GATE）
    ├── lessons-for-v2.md              # v1→v2 设计需求映射（F1-F6）
    ├── lessons-for-v3.md              # v2→v3 设计需求映射（F7-F11）
    ├── extraction-v1/                 # v1 提取基线
    ├── independent-audit-v1/          # v1 独立审计基线
    ├── extraction-v2/                 # v2 重跑结果
    └── extraction-v3/                 # v3 重跑结果（三层知识架构）
```

## 版本演进

| 版本 | Codex 考古结果 | 关键结论 |
|------|---------------|---------|
| v1 | 7 KO（声称 7/7 PASS、0 反例） | 独立审计发现 False Acceptance ≈43%，事实错误 1、过度升维、3 个 Critical Missing |
| v2 | 7 KO（2 L4 + 4 L3 + 1 L2，层级更保守） | 补齐 v1 全部 6 项遗漏；新增 Policy Flow |
| v3 | 8 KO（2 L4 + 6 L3）+ 52 EK + 三层配比 | 三层知识架构（宽底座 + 窄尖顶） |

## 约定

- 每个项目的考古结果按 `benchmarks/<project>/` 归档
- 已完成的提取保留原始版本（extraction-v1/v2/v3），不修订覆盖——保留失败痕迹供 skill 工程回溯
- 本地路径引用已清理，公开内容不含本机目录结构

## License

MIT
