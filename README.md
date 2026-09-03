# knowledge-archaeology-corpus

**已完成项目考古结果的存档库（Corpus）**。保存 knowledge-archaeology skill 对真实项目执行考古后产出的全部知识资产——包括原始提取、独立审计、失败分析、版本演进基线。

与 [knowledge-archaeology-skill](https://github.com/Thy985/knowledge-archaeology-skill) 的分工：
- **skill 仓库**：方法论本身（Multi-Agent Knowledge Archaeology System）
- **corpus 仓库**：用该方法论对具体项目做考古后，沉淀下来的**已完成考古结果**

## 目录约定（每项目一个顶级目录）

```
knowledge-archaeology-corpus/
├── README.md            # 本说明（项目目录约定）
├── LICENSE
└── <project>/           # 每个被考古项目一个顶级目录，如 codex/、tafcm/、formulafix/
    ├── README.md        # 项目说明：考古时间线、版本演进、关键结论（推荐但可选）
    ├── <提取产物>        # 各版本提取（v1/v2/v3 保留原始版本，不修订覆盖）
    ├── validation/      # 独立对抗审计
    └── benchmarks/      # 版本演进基线（failure-analysis / lessons / extraction-*）
```

**添加新项目**：新建 `<project>/` 目录，按 codex/ 内部同构组织即可。

## 当前项目

- [`codex/`](codex/) — OpenAI Codex 多轮考古（v1/v2/v3）

## 约定

- 每个项目的考古结果按 `<project>/` 归档
- 已完成的提取保留原始版本（extraction-v1/v2/v3），不修订覆盖——保留失败痕迹供 skill 工程回溯
- 本地路径引用已清理，公开内容不含本机目录结构

## License

MIT
