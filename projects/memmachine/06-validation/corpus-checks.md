# Corpus Checks — 写入前校验

> 执行时间：2026-11-14（Asia/Shanghai）。对象：`projects/memmachine/`。
> 目的：在 commit 前验证 schema、计数、引用、重复与结构；发现问题改 Corpus Artifact，不改生产 Skill。

| 检查 | 方法 | 结果 |
|---|---|---|
| YAML schema | `python3 yaml.safe_load` 读 `metadata.yaml` / `run-metadata.yaml` | PASS（22 / 16 keys） |
| EK 计数 | 正则 `^### (EK-\d+)` | 52，无重复 |
| EK links | 正则 `(mechanism\|subsystem\|causal\|dependency\|constraint\|contrast):(EK-\d+)` | 145，平均 2.79 |
| EK 目标缺失 | 所有 link target 是否在 52 节点内 | 0 |
| EV 计数 | 正则 `^\| (EV-\d+) \|` | 92，无重复 |
| KO 计数 | 正则 `^## (KO-\d+)` | 8 |
| Flow Edge 计数 | 正则 `^\| (F-[A-Z]-\d+) \|` | 72，无重复 |
| Evidence 引用回链 | EK/KO/Flow/Candidates 中 `EV-*` 是否都在 evidence-index | 0 缺失 |
| EK 引用回链 | EK/KO/Flow/Candidates 中 `EK-*` 是否都在 ek.md | 0 缺失 |
| 项目 slug 重复 | `projects/memmachine` 是否已存在 | 不存在（本轮新建） |
| 历史覆盖 | 是否改写已有 `projects/*` | 未改写；仅新增 `projects/memmachine/` |
| 结构 | 7 类顶层产物 + `archaeology-runs/<run-id>/` | 齐全 |

## 已知非阻断差异

- `1,347 -> 2,191` 是自检修正记录，保留在 `self-validation.md` 的修正表中，不是当前计数。
- `independent-validation.md` 中“与 `package/` 对照”指本地考古 Package，不是 Corpus 路径；保留原文以维持审计上下文。
