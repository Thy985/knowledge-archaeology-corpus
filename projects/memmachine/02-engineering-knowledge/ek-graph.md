# EK Graph — 边、聚合规则与质量指标

> 数据来源：`ek.md` 中的 `**Links**` 行，脚本计数（见本节“复算方法”）。不是估算。

## 1. 规模与质量指标

| 指标 | 值 | 口径 |
|---|---:|---|
| EK 节点 | 52 | `### EK-xx` 计数 |
| 声明 links | 145 | 每条 `type:EK-xx` 计数 |
| 平均出边 | 2.79 | 145 / 52 |
| 最小 / 最大出边 | 2 / 3 | 逐节点 |
| 游离 EK（0 出边） | 0（0.0%） | 要求 <20% |
| 目标节点缺失 | 0 | 所有 target 都在 52 节点内 |
| 有反向边的 link | 88 / 145 | 双向可追溯，不作为强要求 |
| 聚合规则覆盖 | 8 / 8 KO = 100% | 见 §3 |
| 簇规模 | 5 ~ 9 EK；平均 6.5 | 见 §3 |

### 边类型分布

| 边 | 数量 | 语义 |
|---|---:|---|
| `causal` | 54 | 决策/写入/读取的因果链 |
| `contrast` | 30 | 同一问题的两种解法（declarative vs event；写 vs 删） |
| `subsystem` | 23 | 同属一个子系统（STM、semantic、API…） |
| `constraint` | 18 | 边界/前置条件（配置、租户、单位） |
| `dependency` | 11 | 运行时保证依赖（索引依赖原文、检索依赖边） |
| `mechanism` | 9 | 共享实现机制（多路 gather、staged write、refcount） |

### 复算方法

```bash
python3 - <<'PY'
import re
from collections import Counter
txt = open('ek.md', encoding='utf-8').read()
blocks = re.split(r'\n### (EK-\d+)', txt)
eks = {blocks[i]: re.findall(r'`(mechanism|subsystem|causal|dependency|constraint|contrast):(EK-\d+)`', blocks[i+1])
       for i in range(1, len(blocks), 2)}
deg = [len(v) for v in eks.values()]
print(len(eks), sum(deg), sum(deg)/len(deg), Counter(t for v in eks.values() for t, _ in v))
PY
```

## 2. 关键边簇（按子图）

### 2.1 写入路径（declarative vs event 对照）

```
EK-13 (batch order)
   └─causal→ EK-14 (top conditional multi-way write)
        ├─causal→ EK-15 (return_exceptions aggregate)
        │      └─constraint→ EK-16 (TODO no write intent)
        ├─causal→ EK-18 (declarative nodes→edge)
        │      ├─causal→ EK-03 (dual representation)
        │      └─mechanism→ EK-20 (event 5-stage)
        │             ├─causal→ EK-21 (segment before vector)
        │             └─subsystem→ EK-04
        └─contrast→ EK-17 (inner STM/LTM gather, fail-fast)
```

**读法**：两条 backend 写入路径是 `contrast`（同一目标、不同实现），但在“先写权威结构、后写检索索引”上共享 `mechanism`；顶层与内层的并发语义是 `contrast`（等全部 vs 先失败先抛）。

### 2.2 读取路径（单位分歧）

```
EK-32 (declarative: derivative→episode→neighborhood)
   ├─dependency→ EK-18 (needs DERIVED_FROM edges)
   ├─dependency→ EK-34 (context unit)
   └─causal→ EK-36 (threshold)
EK-33 (event: embed→segment→window)
   ├─dependency→ EK-34
   └─causal→ EK-35 (reranker score)
EK-36 ─contrast→ EK-37 (docstring vs declarative impl)
       └─contrast→ EK-38 (REST reintroduces -inf)
```

**读法**：`EK-34` 是最重要的 `constraint` 节点——它把“同一 `expand_context` 参数”在两个 backend 下解释成不同粒度；任何跨 backend 的检索比较都必须先固定这个单位。

### 2.3 状态与删除

```
EK-31 (session delete queue)
   ├─mechanism→ EK-48 (logical delete + worker)
   └─causal→ EK-49 (TaskGroup cancel siblings)
EK-45 (STM eviction) ─causal→ EK-46 (silent summary failure)
EK-47 (instance LRU refcount) ─dependency→ EK-45
EK-50 (shared segment tables) ─subsystem→ EK-04
```

## 3. KO 聚合规则矩阵（R1-R4）

| KO | 规则 | 簇内 EK | 边类型 | 跨实例性（≥2 独立子系统/文件） | 解释范围扩大 |
|---|---|---|---|---|---|
| KO-01 | **R1 机制簇** | EK-03, EK-04, EK-18, EK-20, EK-21, EK-32, EK-33, EK-34, EK-51 | mechanism 3 + contrast 2 + dependency 3 + causal 1 | declarative / event / retrieval / delete 四个子路径 | 从“某后端怎么写”提升为“原文权威 + 派生索引”的一般契约 |
| KO-02 | **R2 因果链簇** | EK-14, EK-15, EK-16, EK-17, EK-18, EK-21, EK-28, EK-51 | causal 5 + contrast 2 + dependency 1 | 顶层编排 / 两个 backend / semantic / 读侧 | 从“某处 gather 没回滚”提升为“多存储写入无补偿的系统性失败模式” |
| KO-03 | **R2 因果链簇** | EK-06, EK-07, EK-08, EK-09, EK-02 | causal 3 + constraint 2 | 配置加载 / 资源门控 / 凭证校验 / backend 选择 | 从“默认 false”提升为“能力可用性由配置完整性决定” |
| KO-04 | **R4 主题簇** | EK-19, EK-32, EK-33, EK-34, EK-35, EK-42, EK-43 | contrast 3 + dependency 2 + constraint 1 + causal 1 | declarative / event / API schema / query formalization | 从“两个实现不同”提升为“上下文单位是检索语义的一等参数” |
| KO-05 | **R3 不变量簇** | EK-05, EK-10, EK-11, EK-43, EK-48 | constraint 2 + mechanism 1 + subsystem 2 | REST / MCP / session key / semantic set | 从“没写 auth”提升为“身份是命名空间约定，不是信任边界” |
| KO-06 | **R2 因果链簇** | EK-22, EK-23, EK-24, EK-25, EK-26, EK-27 | causal 4 + constraint 1 + contrast 1 | semantic service / ingestion / LLM command / citation | 从“异步抽取”提升为“同步原文 + 异步结构化派生的两阶段记忆” |
| KO-07 | **R2 因果链簇** | EK-25, EK-26, EK-27, EK-28, EK-29 | causal 4 + mechanism 1 + contrast 1 | ingestion / consolidation / provenance | 从“合并实现”提升为“破坏性维护 + provenance 膨胀” |
| KO-08 | **R4 主题簇** | EK-31, EK-45, EK-47, EK-48, EK-49, EK-50 | causal 3 + mechanism 2 + dependency 1 + subsystem 1 | STM / instance cache / session / segment store | 从“删除慢/缓存保守”提升为“生命周期是引用计数 + 最终一致清理” |

### 聚合规则自检（防“同子系统=聚合理由”）

- **KO-01** 不是“都属于 episodic memory”：它的共同机制是“权威结构 + 派生索引 + 回填/扩展”，跨越 declarative、event、retrieval、delete 四条路径。
- **KO-02** 不是“都属于写入”：它是一条完整因果链（三路写 → 部分失败 → 无 intent → 读侧漂移），并对照内层 fail-fast 与删除 TaskGroup。
- **KO-03** 不是“都属于配置”：它是“缺一项 → 整层关闭 → 能力静默消失”的因果链，跨配置加载、资源门控、凭证校验。
- **KO-05** 不是“都属于 auth”：它的不变量是“服务端只做命名空间分区，不做信任判定”，跨 REST/MCP/session/semantic 四处一致。
- **KO-08** 不是“都属于 session”：它的共同机制是“refcount + 延迟清理 + 最终一致”，跨 STM、实例缓存、session worker、segment registry。

## 4. 图的结构性风险（观察，不构成 KO）

1. **causal 边偏多（54/145）**：因为系统的主要知识是“写入/读取/删除的顺序”；这符合项目性质，但也意味着 `contrast` 边（30）承担了大部分“设计选择”的解释。
2. **EK-12 / EK-30 / EK-39 / EK-40 / EK-41 / EK-44 / EK-46 未被 KO 直接吸收**：它们是有效的工程知识，但尚未形成足够大的跨子系统簇；按 v3 规则保留在宽底座，不强行升维。
3. **EK-50 只有 S2**：它来自设计文档，不是实现代码；不得与 S3 的 EK 混为同一证据等级。
