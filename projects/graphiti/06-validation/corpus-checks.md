# Corpus Checks — Graphiti `ARCH-2026-11-15-001`

执行位置：`projects/graphiti/`。检查在写入前/提交前执行，问题修正的是 Corpus Artifact，不是生产 Skill。

## 1. Structure Check

| 要求 | 结果 |
|---|---|
| `metadata.yaml` | 存在 |
| `00_overview.md` | 存在 |
| `01-project-layer/` | 存在（1 个文件） |
| `02-engineering-knowledge/` | 存在（1 个文件） |
| `03-knowledge-layer/` | 存在（1 个文件） |
| `04-flow-atlas/` | 存在（1 个文件） |
| `05-candidates/` | 存在（1 个文件） |
| `06-validation/` | 存在（4 个文件） |
| `archaeology-runs/ARCH-2026-11-15-001/` | 存在（完整 run 快照） |
| 空文件 | 0 |
| 文件总数（md/yaml） | 25 |

## 2. Schema / Count Check

| 项 | 结果 |
|---|---|
| EV IDs | 114（EV-001..EV-114），无重复 |
| EK IDs | 58（EK-01..EK-57，含 EK-41b），表行 58，无重复 |
| EK with ≥1 link | 58 / 58 = 100% |
| EK link declarations | 96；平均 96/58 ≈ 1.66 |
| KO IDs | 8（KO-01..KO-08），无重复 |
| KO aggregation rule coverage | 8/8 = 100% |
| Flow edges | 89（Control 19 / State 16 / Data 12 / Evidence 10 / Authority 13 / Memory 10 / Policy 9） |
| Runtime candidates | 20（C-01..C-20） |
| Cross-project candidates | 5（XP-01..XP-05） |
| Needs human review | 7（HR-01..HR-07） |
| S4 / S5 | 0 / 0 |
| Tests executed | 0 |

## 3. Duplicate Check

| 检查 | 结果 |
|---|---|
| Corpus 已有 `projects/graphiti/` | 无（新建） |
| 重复 EV ID | 0 |
| 重复 EK ID | 0 |
| 重复 KO ID | 0 |
| 覆盖上一轮 run | 否；本次为 `archaeology-runs/ARCH-2026-11-15-001/` |

## 4. Evidence Reference Check

| 检查 | 结果 |
|---|---|
| `EV-001..EV-114` 是否全部被引用 | 114 / 114，未引用 0 |
| 引用了不存在的 EV | 0 |
| 引用了不存在的 EK | 0 |
| 引用了不存在的 KO | 0 |
| `run_metadata.yaml` commit | `ed23196506eba8806fae6da831d73fcd2ad714d8` |
| `metadata.yaml` commit | 同上 |
| Package 内 commit | 同上 |

## 5. Metadata Check

| 项 | 值 |
|---|---|
| project / slug | Graphiti / `graphiti` |
| skill / version | `knowledge-archaeology` / `3.2` |
| mode / priority | `initial` / `A-` |
| license | `Apache-2.0` |
| package version | `0.31.2` |
| status | `active` |
| validation | `PASS_WITH_LIMITATIONS` / `PARTIALLY_CONFIRMED` |
| related projects | `mem0`, `memmachine`, `letta-code`, `agent-memory-guard`, `dsh-memory-evolve` |

## 6. Git Check

```
branch: arch/graphiti-2026-11-15
base:   origin/main 9d27bc5
new:    projects/graphiti/**
```

- 只新增 `projects/graphiti/`，未修改其他项目目录。
- 未覆盖既有历史；新 run 独立目录。

## 7. 结果

**PASS**：结构、计数、重复、证据引用、metadata、commit 一致性全部通过。未用弱化证据标准换取全绿；S4/S5 仍为 0。
