# 01 Project Layer — Config / Testing / Governance

## 1. 配置体系

### 1.1 两层配置
| 层 | 文件 | 内容 |
|---|---|---|
| 记忆层 | `EvolveLab/config.py` | `DEFAULT_CONFIG["EvolveLab"]`（`top_k=3`、`active_provider="agent_kb"`）+ 13 provider 段 |
| 进化层 | `MemEvolve/config.py` | 数据集→runner 映射；`ANALYSIS_MAX_STEPS=20`、`CREATIVITY_INDEX=0.5`、`CREATIVITY_TEMP_BASE=0.3`、`CREATIVITY_TEMP_SPAN=0.9`、`EVOLVE_TASK_BATCH_X=20`、`EVOLVE_TOP_T=2`、`EVOLVE_EXTRA_SAMPLE_Y=5`、`EVOLVE_GENERATED_M=3`、`EVOLVE_RANDOM_SEED=42`、`SMOKE_TASK_LIMIT=5`、`IMPORT_TIMEOUT_SEC=10` |

（E-089；`MemEvolve/config.py:16-56`）

### 1.2 覆盖链
```
DEFAULT_CONFIG → provider 构造函数读取 config.get(key, default) → CLI/AutoEvolver 覆盖
```
- 每个 provider 都有**独立默认值**，与 `EvolveLab/config.py` 的段落可能不一致（例如 Evolver 类内 prune 阈值 0.2，config 段 0.3，E-032）。
- `get_config()` 返回浅拷贝（E-010）；嵌套 dict 仍可被外部修改。

### 1.3 重要配置项（已核）
| provider | 配置 | 默认 | 影响 |
|---|---|---|---|
| Lightweight | `enable_longterm_provision` | **False** | 长期记忆“存而不取” |
| Lightweight | `max_shortterm_items` / `shortterm_provision_interval` / `top_k_longterm` | 15 / 5 / 3 | 短期容量与注入频率 |
| Cerebra | `top_k` / text:semantic / `min_score` / `semantic_edge_threshold` / `consolidation_interval` | 5 / 0.2:0.8 / 0.22 / 0.75 / 50 | 图扩展与合并频率 |
| Evolver | `search_top_k` / `prune_threshold` | 1 / 0.3（类内 0.2） | 召回宽度与剪枝 |
| Agent-KB | `top_k` / text:semantic | 3 / 0.5:0.5 | 混合检索 |
| Expel | `top_k` | 1 | 只取一条经验 |

## 2. 测试体系

### 2.1 事实
- `EvolveLab/`：0 个测试文件。
- `MemEvolve/`：0 个测试文件。
- 全仓测试集中在 vendored `mini-swe-agent/tests`（30 个 `test_*.py`，含 `src/minisweagent/models/test_models.py`）。
- 结论：**元进化框架本身没有单元测试/集成测试**；唯一“测试”是 `PhaseValidator` 的运行时快检（E-066~E-070）。

### 2.2 后果（工程判断）
| 缺口 | 风险 |
|---|---|
| 无 Creator 单测 | 非原子写入的失败路径（E-064）不会被 CI 捕获 |
| 无 Validator 单测 | 自动修复开关的默认值矛盾（E-067）长期存在 |
| 无 provider 契约测试 | 13 个 provider 对 BEGIN/IN 的支持差异（E-021~E-023）只能靠人工发现 |
| 无 storage 迁移测试 | JSON 结构变化会静默破坏旧数据（E-090） |

## 3. 治理与权限

### 3.1 谁能改代码
| 行动者 | 权限 | 证据 |
|---|---|---|
| `MemoryCreator` | 写 `EvolveLab/providers/<module>.py`、改 `memory_types.py`、改 `config.py` | E-063 |
| `SWEAgentValidator` | 修隔离副本，并把修复结果 `copy2` 覆盖真实 provider 文件 | E-068 |
| `AutoEvolver` | 重命名/清空 `storage/{provider}` | E-071 |
| 人类 | 通过 CLI 触发；无审批步骤 | E-087 |

**治理结论**：这是一个**无审批的自我修改系统**——LLM 生成的代码可以在无人确认的情况下进入真实 `EvolveLab` 目录（只要验证通过，或被自动修复到通过）。唯一的“门”是 `PhaseValidator`，而它同时承担修复职责。

### 3.2 边界与例外
- 静态检查不是语义检查（E-066）：只要文件里出现方法名/类名文本即可通过。
- 隔离环境只复制 `EvolveLab` + `storage`，不复制 `FlashOAgents`/runner；provider 对运行时的隐式依赖不会被验证（E-069）。
- `IMPORT_TIMEOUT_SEC=10` 仅用于导入（配置存在），未在主流程中看到强制超时实现（候选 C-05）。
- 失败清理缺失：仅 `verdict=passed` 才清理隔离目录（E-070），长期运行会积累副本。

## 4. 文档与实现的偏差（已核）
| 文档声明 | 实现 | 判定 |
|---|---|---|
| `PhaseValidator` “static checks ... without automatic fixes”（docstring） | 构造函数 `enable_auto_fix=True`、`max_fix_attempts=3`，且会 sync 回真实目录 | **矛盾**（E-067/E-068） |
| README “11 baselines + 2 evolved = 13” | 代码只有 13 个 enum/mapping，无 “evolved” 标记 | 文档声明，不可当实现事实（E-093） |
| README “not the final release” | 与仓库状态一致 | S0-vendor（E-094） |
