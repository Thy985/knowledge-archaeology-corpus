# 06 Validation / Independent Validation（ARCH-2026-11-16-001）

## 0. 方法与边界
- **Independent Auditor 输入**：目标仓库 `b63f76eece80d84387b6056919bd94816c60bd89` 的源码、测试、文档；不使用生产包作为事实来源。
- **盲重建方式**：先独立重读 `guard.py` / `identity.py` / `policies/access.py` / `integrations/llamaindex.py` / `server.py` / `storage/snapshots.py` / `classification.py`，再用独立运行时探针（`PYTHONPATH=src`）复现行为；完成独立记录后才与 `00–05` 对比。
- **限制**：审计员在同一会话中见过项目名与旧 corpus，不能宣称“完全无先验”；因此把“源码证据 + 独立探针”作为事实门槛，包内文字只用于对比，不作为证据。
- **未覆盖**：真实 `llama-index-core` / `agno` 未安装；未启动 HTTP server 做端到端；未复跑 benchmark；未核验 PyPI 构建 provenance。

## 1. 判定统计
| 判定 | 数量 | 说明 |
|---|---|---|
| CONFIRMED | **26** | 独立源码/探针与包内断言一致 |
| PARTIALLY_CONFIRMED | **5** | 机制成立，但范围/强度被削弱 |
| DOWNGRADED | **0** | 无“必须降层”的 KO；KO-07 已自带 L4 candidate 限定 |
| OVER_GENERALIZED | **0** | 无跨项目 Principle 被写成已验证 |
| CONTRADICTED | **4** | 包内初版计数/schema/方法/版本归属错误，已在 Reconciliation 修正 |
| MISSING | **5** | 独立发现但包内未记录的机制/边界 |
| NEEDS_HUMAN_REVIEW | **0** | 争议均可在包内分层处理，未升级为人工决策 |
| **合计** | **40** | 对应 IF-01…IF-40 |

## 2. 独立发现表（IF-01…IF-40）

### 2.1 访问控制 / 身份（IF-01…IF-10）
| ID | 独立发现 | 证据 | 判定 |
|---|---|---|---|
| IF-01 | 默认 guard 注册 7 个 detector，名称与包内一致 | 探针 `len(_detectors)=7`；`guard.py:150-163` | CONFIRMED |
| IF-02 | `AccessPolicy.decide()` 顺序 registry → admin → first rule → default → class | `policies/access.py:445-544` | CONFIRMED |
| IF-03 | 未声明 principal `bob` 被拒，错误为 `unknown principal 'bob'` | 探针 | CONFIRMED |
| IF-04 | 无 admins 时 `snapshot()` 被拒，错误明确 “policy names no admins” | 探针 | CONFIRMED |
| IF-05 | class gate 同时检查 current/target，且在 key rule 之后 | `access.py:489-518`；`test_class_gate_on_promote_target` | CONFIRMED |
| IF-06 | read gate 先于存在性/完整性，missing key + outsider 返回 `AccessDenied` 而非默认值 | 探针；`guard.py:950-977` | CONFIRMED |
| IF-07 | 被拒 write 不落 store、不产生 quarantine 项 | 探针 `'k' in _store.keys()` False；`quarantine=[]` | CONFIRMED |
| IF-08 | `AgentHandle` 把 principal 固定为 `_pid`，禁止 `principal=` 覆写 | `identity.py:568-648` | CONFIRMED |
| IF-09 | `AgentHandle` 不暴露 events/quarantine/snapshots/classify/verify_all | `identity.py` 方法表 | CONFIRMED |
| IF-10 | raw guard 仍暴露上述内省接口，且 `list_snapshots` 不受 access gate | 探针：无 admins 的 guard 返回 Snapshot 列表 | CONFIRMED |

### 2.2 检测 / 处置（IF-11…IF-18）
| ID | 独立发现 | 证据 | 判定 |
|---|---|---|---|
| IF-11 | REDACT 对 dict 会先 `_stringify` 再替换，read 返回 `str`；原始结构丢失 | 探针：`'token: [REDACTED:github_token]\nn: 1'` | CONFIRMED |
| IF-12 | `verified_by` 仅写入 event metadata，不做身份认证；伪造值可随 promotion 记录 | 先 `cls=USER_PREFERENCE_CANDIDATE`，再 promote → event `verified_by='not_a_real_admin'` | CONFIRMED |
| IF-13 | `promote(key, "verified_preference")` 在无 access 时抛 `AttributeError`，有 access 时归一化 | 探针；`guard.py` promotion 分支 | CONFIRMED |
| IF-14 | `_run_detectors()` 单 detector 异常 fail-open，丢 verdict 并发 LOW ALLOW 事件 | `guard.py:1338-1357` | CONFIRMED |
| IF-15 | `events` 是 property，返回 `list(self._events)` 拷贝；`_events` 无容量上限 | 源码；探针 `len(events)=1` | CONFIRMED |
| IF-16 | `quarantine` 是 dict，无容量上限；read 路径不查询 quarantine | `guard.py`；`_read_impl` | CONFIRMED |
| IF-17 | SnapshotStore 容量 50，内部字段 `_max`，不是 `max_snapshots` | 探针 `__dict__={' _max':50,...}` | CONFIRMED |
| IF-18 | 默认 `SensitiveDataDetector` 忽略 email；redact 仍按 detector 名找同名 detector | `detectors/leakage.py` 默认参数 | CONFIRMED |

### 2.3 适配器 / 服务边界（IF-19…IF-24）
| ID | 独立发现 | 证据 | 判定 |
|---|---|---|---|
| IF-19 | `GuardedChatStore.get_messages()` 直读底层，不跑 access/detector/redact | `integrations/llamaindex.py:88-93` | CONFIRMED |
| IF-20 | outsider 的 `guard.read` 被拒，但 `store.get_messages('owner')` 返回 `['SECRET']` | 独立 fake backend 复现 | CONFIRMED |
| IF-21 | `store.guard.read(..., principal='owner')` 可冒充 owner，返回 `SECRET` | 独立复现 | CONFIRMED |
| IF-22 | LlamaIndex 测试把该读旁路当既定行为：denied-delete 测试只断言 delete 被拒，同时断言内容可见 | `tests/test_integration_llamaindex.py:47-78` | CONFIRMED |
| IF-23 | `server.py` 明确无认证；`/write`、`/read` 不传 principal；`/reset` 无认证 | `server.py:35-52,114-170,264-339,426-431` | CONFIRMED |
| IF-24 | CORS 默认关闭，`AMG_CORS_ORIGINS=*` 直接 `SystemExit`，但这不是认证 | `server.py` CORS 段 | CONFIRMED |

### 2.4 文档 / 版本 / 指标（IF-25…IF-30）
| ID | 独立发现 | 证据 | 判定 |
|---|---|---|---|
| IF-25 | `design.md` 列 10 detector、4 类 Classification、read 先查 quarantine；实现是 7 detector、6 类、read 不查 quarantine | `docs/architecture/design.md` vs `guard.py` / `classification.py` | CONFIRMED |
| IF-26 | README/compliance 的 92.5% recall/100% precision/62 µs 仍 pin 0.2.2，不是 0.4.0 | `README.md`、`docs/compliance-mapping.md`、`benchmarks/results/benchmark_report.md` | CONFIRMED |
| IF-27 | `v0.4.0` tag `b8353d6` 包含安全修复；release commit `5a2ac20` 早于修复；CHANGELOG 仍列 `[Unreleased]` | `git merge-base --is-ancestor 15b75be b8353d6` exit=0；`git show 5a2ac20` | CONFIRMED |
| IF-28 | 当前 `pyproject` 仍为 0.4.0，tag/release/changelog 三者不一致 | `pyproject.toml`、`CHANGELOG.md`、tag | CONFIRMED |
| IF-29 | 全量测试 504 passed / 2 skipped / 10 xfailed；2 skip 为 agno/llama_index 缺失 | `pytest -q` 本轮执行 | CONFIRMED |
| IF-30 | 10 xfail 是项目自报未实现攻击族，不是“测试通过” | `tests/benchmarks/test_delayed_activation.py` 等 | CONFIRMED |

### 2.5 知识层判定（IF-31…IF-35）
| ID | 独立发现 | 证据 | 判定 |
|---|---|---|---|
| IF-31 | KO-01（authorization before effects）成立：gate 在 body/检测器/快照之前 | EK-01/07/08/20 + 探针 | CONFIRMED |
| IF-32 | KO-02（integrity at write, not at rest）成立；digest 不校验、baseline 只在首写 | EK-21/40；`integrity.py` | CONFIRMED |
| IF-33 | KO-03（fail-open but visible）只部分成立：事件可观测，但无消费者、无上限，且失败不阻断 | EK-38/39；探针 | PARTIALLY_CONFIRMED |
| IF-34 | KO-04（trust transition as graph）只部分成立：图与 class gate 存在，但 `verified_by` 可伪造、target 归一化不一致 | EK-41/53/54 | PARTIALLY_CONFIRMED |
| IF-35 | KO-05（scope = weakest equivalent path）只部分成立：Llama 旁路强证据，但 LangChain 读走 guard，不能写成“所有适配器都绕过” | EK-42/45；KO-05 反例 | PARTIALLY_CONFIRMED |

### 2.6 包内错误（IF-36…IF-40）
| ID | 独立核对 | 包内初版 | 判定 |
|---|---|---|---|
| IF-36 | EK Graph 出现 `boundary` 伪边（EK-17/23/24/28/29/30/47/50 共 7 处） | 初版 `02` 有 `boundary` | CONTRADICTED（已修正为 `contrast`/`constraint`） |
| IF-37 | 实际 EK=55，不是 48 | `00` 写 “EK-01…EK-48” | CONTRADICTED（已修正） |
| IF-38 | 实际 C=15，不是 14 | `00` 写 “C-01…C-14” | CONTRADICTED（已修正） |
| IF-39 | `events` 是 property；初版写 `events()` 方法 | EK-14/38/55 初版 | CONTRADICTED（已修正） |
| IF-40 | KO-07 的 L4 只是 candidate，不应按 L4 已验证计 | `03` 标 “L4 候选” | PARTIALLY_CONFIRMED（无需降级） |

## 3. 三个独立成功（高价值确认）
### S-01 读门先于存在性，拒绝不可被探测（IF-06）
独立复现：
- `read('missing', principal='bob')` → `AccessDenied: read of 'missing' denied: unknown principal 'bob'`。
- 不是 `KeyError`、不是默认值。
**意义**：E 层“拒绝不泄露存在性”是真实实现，不是文档口号；可作为 11-20 作用域串读实测的正向基线。

### S-02 LlamaIndex 读路径完全绕过 ACL（IF-19/20）
独立复现：
- owner 通过 `GuardedChatStore.set_messages('owner', ['SECRET'])` 写入；
- outsider 的 `guard.read('llamaindex_messages.owner.0', principal='outsider')` → `AccessDenied`；
- 同一 outsider 调 `store.get_messages('owner')` → `['SECRET']`。
**意义**：这是本轮最强 E 层反向证据；“写门有效、读门可绕”是可执行事实（fake backend，S5）。

### S-03 `verified_by` 只是事件元数据（IF-12）
独立复现：
- 先把 key 写成 `USER_PREFERENCE_CANDIDATE`；
- `promote(..., verified=True, verified_by='not_a_real_admin')` 成功；
- event metadata 记录 `verified_by='not_a_real_admin'`。
**意义**：KO-04 的“信任图”不能被当作“审计归属已验证”；这是 audit authority 的边界，不是 promotion 图不存在。

## 4. 三个独立错误（包内初版）
### E-01 schema 伪边：7 处 `boundary`（IF-36）
`boundary` 不在 EK Graph 六类边中；它不是“对照/约束”的同义词。独立审计按 schema 直接判 `CONTRADICTED`。
**修正**：EK-17/23/24/28/29/30/47/50 改为 `contrast` 或 `constraint`；重算 166 edges / 3.02。

### E-02 计数不一致：EK 48 vs 55、C-14 vs 15（IF-37/38）
初版 `00_overview` 的 48/2.10/C-14 是旧口径或预填，不是脚本计数。独立重算 EK=55、edges=166、avg=3.02、C=15。
**修正**：00 与 02 文末统计同步为 55/166/3.02/15。

### E-03 方法名与版本归属（IF-39/27）
- `events` 是 property，不是 `events()`；初版三处写成方法。
- 安全修复不是“0.4.0 发布内容”，而是 release 之后进入 main、当前 tag 包含、CHANGELOG 仍列 `[Unreleased]` 的修复。
**修正**：EK-14/38/55 改 property；EK-27 改为 post-release 精确表述；C-15/HR-05 保留版本归属争议。

## 5. 五个遗漏（MISSING）
### M-01 `set_messages` 的 guard key 与 backing index 可能错位
`set_messages` 用 `enumerate(messages)` 生成 `key.i`，但 blocked/quarantine 的消息被 `continue` 跳过；backing store 只收到 `screened` 列表。若第 0 条被拦、第 1 条通过，guard 存 `key.1`，backing 的第 0 条是原第 1 条；之后 `add_message` 用 `len(backing)` 计算 position，可能再次写 `key.1`，覆盖/错配。
**证据**：`integrations/llamaindex.py:75-93, 95-109`。
**影响**：读回的消息与 guard 的 key/verdict 可能不对应；属于数据一致性缺陷，不是纯安全绕过。
**验证路径**：fake backend + 第一条 BLOCK、第二条 ALLOW；检查 guard keys 与 backing 索引。

### M-02 `retire_if` 的 predicate 异常被吞掉并跳过 key
`guard.py:1170-1173`：predicate 抛异常 → `log.exception` + `continue`；该 key 既不 retire 也不上报失败。admin 可能以为批量退休已完成。
**证据**：`guard.py:1155-1195`。
**影响**：批量治理的“部分成功”没有回执，和 delete 回执一致性主线相关。

### M-03 `server /read` 的 `found=value is not None` 把“存了 None”当成“不存在”
`server.py:330-336`：`found=value is not None`；若合法写入 `None`，read 返回 `found=False`，与缺失不可区分。
**证据**：`server.py:312-336`；`write` 允许任意 value。
**影响**：HTTP 客户端无法区分“空值”与“未写入”，属于回执语义问题。

### M-04 `set_messages` 全被拦截时不会清空旧消息
`if screened: self._store.set_messages(...)`；若新列表全部 BLOCK/QUARANTINE，底层旧消息保持原样。调用方以为“已设置新列表”，实际读到旧内容。
**证据**：`integrations/llamaindex.py:75-87`。
**影响**：与 M-01 同属适配器状态一致性；可能造成“删除/覆盖失败但无回执”。

### M-05 0.3.3 → 0.4.0 的旧 KO 差异表缺失
旧 corpus 有 KO-01…KO-08（决策管线、不可检查=发现、fail-open 可见、分层防线族、信任梯度晋升图、诚实边界声明、事件观测闭环、可判定安全基准）。本轮新 KO-01…KO-10 未逐条标注“保留 / 修订 / 失效 / 新增”。
**证据**：`projects/agent-memory-guard/03-knowledge-layer/knowledge_layer.md`（旧）vs 本轮 `03`。
**影响**：refresh 的认知连续性不足；读者无法判断旧结论是否仍成立。

## 6. Benchmark Case
### BC-01：AMG 0.3.3 → 0.4.0 refresh 回归
- **输入**：旧包 `ARCH-2026-10-07-001`（0.3.3，8 KO）+ 当前仓库 `b63f76e`（0.4.0）。
- **期望**：refresh 必须输出旧 KO 的逐条状态（保留/修订/失效）与新 KO 的增量关系；旧 EK 的失效项要标注。
- **实际**：新包有 10 KO、55 EK，但没有旧 KO 对照表；只在 `00` 声明“旧包继续作为历史快照”。
- **判定**：**FAIL（coverage）**——不是事实错误，而是 refresh 连续性与回归基准缺失。
- **修复建议（skill-level）**：refresh 模式强制产出 `06_validation/refresh-delta.md`，字段：`old_ko_id / old_claim / new_status / superseded_by / evidence_change`。该建议进入 Skill Evolution 的 P2 记录，不在本轮修改生产 Skill。

## 7. 结论
- 本轮独立审计**没有发现新的事实性错误**（除包内初版计数/schema/方法名/版本归属，均已修正）。
- 最重要的独立增量是 **M-01～M-04**：适配器状态一致性与批量治理回执，属于 E 层（作用域）与 delete 回执主线的直接输入。
- **KO-03/KO-04/KO-05 保持 L3，不升 L4 已验证**；KO-07 的 L4 仅为 candidate。
- **无跨项目 Principle**；所有 L3 均标 `Cross-project validation pending`。
