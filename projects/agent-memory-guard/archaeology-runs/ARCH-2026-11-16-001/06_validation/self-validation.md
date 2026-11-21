# 06 Validation / Self-Validation（ARCH-2026-11-16-001）

> 本文件是**生产侧自检**，不是 Independent Validation。独立 Auditor 的盲重建结果见 `independent-validation.md`；两者冲突在 `reconciliation.md` 中裁决。

## 1. 自检结论
**PASS WITH LIMITATIONS**——结构、计数、边 schema、Flow 数、KO 聚合规则、证据行覆盖率均通过；但存在 4 类限制（真实框架依赖缺失、0.4.0 benchmark 未复跑、Python 多版本未验证、发布 provenance 未核验）。

## 2. 计数与结构门（脚本重算，非估算）
| 检查项 | 结果 | 口径 / 命令 |
|---|---|---|
| EK 节点数 | **55** | `re.findall(r'^### EK-\d{2}', 02, re.M)` |
| EK links 行数 | **55/55** | 每条 EK 恰一行 `links` |
| EK 边总数 | **166** | 按 `；` 切分并逐段匹配六类边 |
| 平均出边 | **3.02** | 166 / 55 |
| 六类边覆盖 | **6/6** | causal 36 / constraint 23 / subsystem 8 / mechanism 21 / dependency 26 / contrast 52 |
| 非法边类型 | **0** | 已把 7 处 `boundary` 伪边改为 `contrast`/`constraint` |
| 游离 EK | **0** | 每条 EK ≥2 条边 |
| KO 数 | **10** | `^## KO-\d{2}` |
| KO 聚合规则覆盖 | **10/10** | 每个 KO 都有 `aggregation_rule` R1–R4 |
| Flow Edge 数 | **49** | `^\| (CT\|ST\|DF\|EV\|AU\|MF\|PF)-\d{2} \|` |
| Flow 七类覆盖 | **7/7** | Control 8 / State 7 / Data 8 / Evidence 7 / Authority 7 / Memory 6 / Policy 6 |
| Candidates | **19** | C-01…C-19（C-16…C-19 为独立验证新增） |
| Cross-project | **5** | XP-01…XP-05 |
| Human Review | **6** | HR-01…HR-06 |
| EK 证据行覆盖 | **55/55** | 每条都有 `证据` 且含文件/符号/测试 |
| KO 回链 | **10/10** | 每个 KO 有 `可回溯` 并指回 EK |

## 3. Schema 检查
- **EK Graph schema**：边类型只在 `mechanism/subsystem/causal/dependency/constraint/contrast`；无 `boundary` 等伪边；每条 EK 有 `层/状态`；无孤立节点。
- **KO schema**：每个 KO 有 `aggregation_rule`、`簇内 EK`、`L1/L2/L3`、`反例`、`Epistemic`、`可回溯`；L3+ 均带 `Cross-project validation pending` 或明确单项目限定；**无 L5**。
- **Flow schema**：每条 Edge 有 `From → To`、`触发/条件`、`证据`；条件为代码分支而非文档描述；49 条与七类计数一致。
- **Candidate schema**：每条有 `类型`、`已有证据`、`缺失证据`、`验证路径`、`Epistemic`；没有把 Hypothesis 写成 Fact。
- **Metadata schema**：`run_metadata.yaml` 字段与 SOP 要求一致（run_id / repository / commit / skill_version / timestamp / quality_metrics / validation_result）。

## 4. 证据链抽检（10 条，全部通过）
| 断言 | 回链 | 结果 |
|---|---|---|
| 默认 7 detector | `guard.py:150-163`；独立探针 `len(_detectors)=7` | PASS |
| unknown principal 拒绝 | `AccessPolicy` 行为探针 → `AccessDenied: unknown principal 'bob'` | PASS |
| 无 admins 不能 snapshot | 行为探针 → `AccessDenied: nobody may snapshot` | PASS |
| read gate 先于存在性 | `missing` key + outsider → `AccessDenied`（非 default/KeyError） | PASS |
| 被拒 write 不落 store | `'k' in _store.keys()` → False；quarantine 空 | PASS |
| REDACT 容器变 str | dict → `'token: [REDACTED:github_token]\nn: 1'` | PASS |
| `verified_by` 仅 metadata | `not_a_real_admin` 进入 event metadata，promotion 成功 | PASS |
| `promote` 字符串 target 崩溃 | `AttributeError: 'str' object has no attribute 'value'` | PASS |
| Llama `get_messages` 旁路 | outsider `guard.read` 拒；`get_messages` 返回 `['SECRET']` | PASS |
| `.guard` 伪造 principal | `store.guard.read(..., principal='owner')` 返回 `SECRET` | PASS |
| `list_snapshots` 无 gate | 无 admins 的 guard 调 `list_snapshots()` 返回 Snapshot 列表 | PASS |

## 5. 本轮发现并已修正的自检错误
1. **00_overview 预填数字**：原写 “EK 48 / 116 links / 2.11 / Flow 49 / Candidates C-14”，实际 EK 55、links 166、avg 3.02、C-15；已按脚本结果修正。
2. **7 处 `boundary` 伪边**：EK-17/23/24/28/29/30/47/50 中的 `boundary` 不是 schema 允许类型；已改为 `contrast` 或 `constraint`，并重算边数。
3. **`events()` 方法误写**：`events` 是 property，不是方法；已在 EK-14/38/55 修正。
4. **安全修复版本归属**：原文称“0.4.0 的安全修复”；实际 CHANGELOG 在 `[Unreleased]`，release commit `5a2ac20` 早于修复，当前 tag `v0.4.0`（`b8353d6`）已包含修复；已在 EK-27 与 C-15 精确表述。
5. **`verified_by` 探针初版用字符串 target**：导致 `AttributeError` 掩盖真实结论；改用 `MemoryClass.VERIFIED_PREFERENCE` 后确认 promotion 成功且 metadata 记录伪造值。

## 6. 反例检查（自检层，非独立）
| KO | 反例 | 是否推翻 |
|---|---|---|
| KO-01 | admin 操作先于 key rule（EK-03） | 不推翻：模式约束“先于副作用”，不是“所有规则同序” |
| KO-02 | `redact()` 成功但结构退化（EK-28） | 不推翻：属于“处置成功但语义损失”，是独立缺陷 |
| KO-03 | fail-open 事件无消费者（C-13） | 削弱“可见”但不推翻“fail-open + 事件”机制 |
| KO-04 | `verified_by` 可伪造（EK-54） | 削弱“审计归属”，不推翻“图 + 门禁” |
| KO-05 | LangChain 读走 guard（EK-45） | 不推翻：说明边界部分闭合，不是全开放 |
| KO-06 | 无 access 的 0.3 兼容路径仍全开（EK-06） | 不推翻：边界是“是否声明 access 意图” |
| KO-07 | `_guard` 可直接访问（EK-16） | 削弱“强制隔离”，支持“能力裁剪≠认证” |
| KO-08 | `AgentHandle` 不暴露内省接口（EK-14） | 不推翻：说明设计者知道风险，但未覆盖 raw guard |
| KO-09 | 504 tests 通过（EK-52） | 不推翻：只覆盖测试集，不覆盖 xfail |
| KO-10 | 项目自述“not adaptive adversary”（KO-10 反例） | 不推翻，反而是正面边界声明 |

## 7. 未覆盖 / 限制
- **真实 LlamaIndex**：未安装 `llama-index-core`；旁路在 fake `BaseChatStore` 上复现，真实框架的消息对象/`ChatMemoryBuffer` 行为未验证。
- **真实 Agno**：未安装；相关测试 skip。
- **Benchmark**：未复跑 0.4.0 的检测率/延迟；README 的 92.5%/62 µs 仍 pin 0.2.2。
- **Python 版本**：只在 3.12 跑；3.9–3.11、3.13–3.14 未验证。
- **PyPI provenance**：未核验 0.4.0 构建的 commit；tag/release/changelog 关系已记录为 C-15/HR-05。
- **真实 HTTP 服务**：未启动 server 做端到端请求；`/reset` 与无认证结论来自代码路径。
- **旧 KO 对照**：未逐条对照 0.3.3 旧包的 KO-04/KO-05/C-04 是否因 0.4.0 失效；只在 00/03 中建立新 KO，未做旧 KO 差异表。

## 8. 交付前自检
- [x] 三层配比：55 EK / 10 KO / 166 edges / 49 Flow / 19 C / 5 XP / 6 HR
- [x] EK Graph 平均出边 3.02 ≥1；游离 0；聚合规则 100%
- [x] 无“同子系统=聚合理由”的假聚合（KO-05 用 mechanism 边 + 跨三处独立位置；KO-08 跨 events/snapshots/raw-guard）
- [x] Hypothesis 在 05，不进入 03
- [x] 未机械复制文档；design.md 漂移作为 EK-48 记录
- [x] 默认交付 corpus；未写飞书（无授权）
- [x] 目标仓库 / KnowlegeMap / 生产 Skill 未修改
