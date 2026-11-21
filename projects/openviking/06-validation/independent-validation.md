# 06 · Independent Validation（盲重建）

> 方法：**不把 Package 当事实来源**。先独立重读仓库（`openviking/storage/viking_fs/_ops.py`、`service/fs_service.py`、`core/namespace_policy.py`、`session/memory/memory_updater.py`、`retrieve/*`、`server/auth/plugins/trusted.py`、`privacy/scan.py`），建立 Independent Findings（IF），再与 Package 对比。
> 禁止修改原考古产物；本报告只给判定与修正建议。

## 1. Independent Findings（15 条）

| IF | 独立发现 | 证据 | 与 Package 关系 |
|---|---|---|---|
| **IF-01** | `rm` 先 `vector_store.delete_uris` 再 `_async_agfs.rm`；两处分支一致 | `storage/viking_fs/_ops.py:240+`（rm 实现段） | 对应 EV-36；**CONFIRMED** |
| **IF-02** | 存在 `no_vector=True` 的删除 bypass：`fs_service._rm` 在 `NamespaceProcessingMode.FILE_ONLY` 时跳过向量清理 | `service/fs_service.py:651-657` | **Package MISSING**；影响 KO-01 边界 |
| **IF-03** | `FILE_ONLY` 仅适用于 `viking://system/...`；该命名空间同时 `supports_acl=False`、`supports_tags=False`、`supports_reindex=False` | `core/namespace_policy.py:35-47, 84-90` | **Package MISSING**；证明 bypass 被严格 scope 限定 |
| **IF-04** | `apply_operations` 在批内存在 unresolved upsert 时**整批跳过 delete**（`has_unresolved_upserts`），不只是 `resolution_skip` 逐条 | `session/memory/memory_updater.py:1180-1320`（删除循环前） | 对应 EK-14/EK-16；Package 描述**不完整**，实际守卫更强 |
| **IF-05** | 默认 tier=abstract；`DEPTH_CEILING_BY_CATEGORY={"events":"full"}`；`FULL_BODY_ABSTRACT_CATEGORIES` 含全部 memory 类 | `retrieve/context_assembler/params.py:56-82` | **CONFIRMED** |
| **IF-06** | trusted 模式每个请求强制 `X-API-Key == root_api_key`（`hmac.compare_digest`）；header 与 URL 不一致拒绝 | `server/auth/plugins/trusted.py:139-172` | **CONFIRMED** |
| **IF-07** | `HierarchicalRetriever.retrieve` 单次 embedding + 单次向量检索 + 可选 rerank（候选 ×2），无递归树遍历 | `retrieve/hierarchical_retriever.py:102-240` | **CONFIRMED** |
| **IF-08** | `mv` 的"复制→更新索引→删源"只在 `09-transaction.md` 有完整描述；Python 侧未见逐行对应实现（底层在 Rust） | `docs/en/concepts/09-transaction.md` | **PARTIALLY_CONFIRMED**（doc-only） |
| **IF-09** | ACL 的 `effective = direct ∪ (restricted ? ∅ : inherited)`、不能回 none、copy 不带源 ACL 均只在文档；代码只证字段与 `is_acl_uri` | `15-acl.md`；`storage/acl.py:51` | **PARTIALLY_CONFIRMED**（doc-only 语义） |
| **IF-10** | `_rm` 先 `_ensure_access(MANAGE)`，随后 `delete_ctx = replace(ctx, bypass_acl=True)` 供内部递归删除使用 | `service/fs_service.py:641-642` | **Package MISSING**；不是漏洞（预检后 bypass），但需记录 |
| **IF-11** | 隐私 `scan` 是**修正式扫描**，有硬限流：1000 节点、1MB/文件、16MB/总量、60s、20 files/s、单进程 1 并发 | `privacy/scan.py:28-45` | **Package MISSING**；支持 C-04（非实时通用 PII 检测） |
| **IF-12** | `experiences` 的 prompt 明确要求 Situation/Approach/Reflect、去除实体、保留 IF/THEN/ELSE、Approach ≤8 bullets、单意图拆分 | `system_assets/.../experiences.yaml` | 对应 EK-20；**CONFIRMED** |
| **IF-13** | 未见 trusted 模式的来源 IP / mTLS 校验代码；安全性依赖 root key 保管与部署网络 | `trusted.py`（无来源校验） | 对应 EK-34；**NEEDS_HUMAN_REVIEW** |
| **IF-14** | `agent_evolution_enabled=False` 时从允许类型剔除 `cases/trajectories`，返回 `reason=agent_evolution_disabled` | `session/compressor_v3.py:425+`；`constants.py:9-13` | **CONFIRMED** |
| **IF-15** | `rm` 对不存在目标走 `_finish_missing_delete`（清孤儿索引），docstring 明写 idempotent | `viking_fs/_ops.py` docstring + missing 分支 | **CONFIRMED** |

## 2. 判定统计

| 判定 | 数量 | 条目 |
|---|---:|---|
| **CONFIRMED** | 7 | IF-01, IF-05, IF-06, IF-07, IF-12, IF-14, IF-15 |
| **PARTIALLY_CONFIRMED** | 3 | IF-04（守卫描述不完整）、IF-08（mv doc-only）、IF-09（ACL 语义 doc-only） |
| **DOWNGRADED** | 1 | KO-02（L4 → 建议 L3，scope 收窄；理由见 §3 E-02） |
| **OVER_GENERALIZED** | 1 | KO-07（"写乐观读保守"被当作模式，实际含 2 条真反例） |
| **MISSING** | 3 | IF-02（no_vector bypass）、IF-03（system-only 限定）、IF-10/IF-11（rm bypass_acl / privacy 限流） |
| **CONTRADICTED** | 0 | 未发现与代码直接矛盾的 claim |
| **NEEDS_HUMAN_REVIEW** | 1 | IF-13（trusted 模式来源校验/部署责任） |
| **合计** | **15** | — |

## 3. 三个成功（独立审计确认 Package 做对了）

1. **S-01 · 删除顺序与失败方向**
   - 独立重读 `_ops.py` 确认：recursive 与 single 两条分支都在 `vector_store.delete_uris` 之后才 `_async_agfs.rm`；missing 分支清孤儿索引。
   - Package 的 KO-01/EV-36 与代码一致，未夸大。
2. **S-02 · 检索不是"层级遍历"**
   - 独立重读 `hierarchical_retriever.py`：一次 `embed_query`、一次 `search_in_tenant`、按 URI 去重、rerank 仅非 image；无 tree walk。
   - Package 明确指出"命名与机制反差"，这是本轮最有价值的纠偏之一。
3. **S-03 · trusted 模式不是裸信任 header**
   - 独立重读 `trusted.py`：root key 强制 + `hmac.compare_digest` + URL/header 一致性校验 + role 白名单。
   - Package 的 EK-33 准确；EK-34 的"root key = impersonation"也被代码结构支持。

## 4. 三个错误（Package 需要修正）

### E-01 · KO-07 过度概括（OVER_GENERALIZED）
- **Package 原述**："写入路径乐观执行，读取路径保守降级"，并试图上升为 L3 模式。
- **独立证据**：`compressor_v3.py` 在 `strict_extract_errors=False` 时静默返回空（写路径，但**不可观测**）；`hierarchical_retriever` 在 collection 缺失时返回空（读路径**静默**）；而 `intent_analyzer` 硬抛错（读路径**不保守**）。
- **判定**：该"模式"不能同时解释三处行为；它更像"按可观测性/可重试性分配失败策略"的**局部观察**。Package 已自行标 `internal tension` 并降 value B，但仍保留为 L3，**建议降为 L3 的 "Candidate Pattern" 或移入 Candidates**。
- **修正**：KO-07 → `Pattern (disputed)`，并在 KO 索引中标注"不用于跨项目推广"。

### E-02 · KO-02 的 L4 层级过高（DOWNGRADED）
- **Package 原述**：L4"不可逆操作应由成功的前置状态授权"。
- **独立证据**：该原则只在 **memory 写入路径**（`apply_operations`）有完整代码证据；`add_resource` 的覆盖/清理路径未验证（C-01），`mv` 只有文档。L4 要求"跨多个独立路径成立"，当前只有 1 条完整路径 + 1 条 doc-only。
- **判定**：**降为 L3**；L4 断言保留在 Candidates（C-05），待跨项目验证。
- **修正**：KO-02 `knowledge_layer` 与标题层级改为 L3；EPISTEMIC 改为 `Pattern (single-path)`。

### E-03 · EK-16 描述不完整（PARTIALLY_CONFIRMED）
- **Package 原述**："unresolved upsert 带 `resolution_skip` 时报告型跳过"。
- **独立证据**：`apply_operations` 在 `has_unresolved_upserts` 为真时，**删除循环整体跳过**并记录 `"Skipped delete because batch contains unresolved upsert URIs"`。这是**批级守卫**，强于 Package 描述的"逐条跳过"。
- **判定**：EK-16 需补一句"批级 unresolved → 全部 delete 跳过"；EK-14 的"六类守卫"实际应为"七类"（新增批级 unresolved 删除守卫）。
- **修正**：EK-14 守卫数 6→7；EK-16 改写为"批级 + 条级双层跳过"。

## 5. 遗漏（Package 未覆盖但真实存在）

### M-01 · `no_vector` bypass 与 system namespace
- `fs_service._rm` 在 `FILE_ONLY` 时调用 `viking_fs.rm(..., no_vector=True)`，跳过向量清理；`FILE_ONLY` 仅 `viking://system/...`。
- **影响**：KO-01 的"索引不领先于源"必须加边界：**system namespace 不参与索引**，因此其删除不适用该不变量。Package 未写，属 Coverage 遗漏。
- **修正**：KO-01 增加 `scope: indexed namespaces only (viking://system excluded)`。

### M-02 · `bypass_acl` 的内部使用
- `_rm` 先做 MANAGE 校验，再以 `bypass_acl=True` 构造 `delete_ctx`，避免递归删除时重复检查。
- **影响**：不是漏洞，但 Authority Flow 缺一条 Edge；需补 `FA-09: precheck MANAGE → bypass_acl for internal recursion`。

### M-03 · 隐私扫描的硬限流
- `privacy/scan.py`：1000 节点 / 1MB 单文件 / 16MB 总量 / 60s / 20 files/s / 单进程 1 并发。
- **影响**：EK-35 应补"隐私保护是写时注册值 + 有限离线扫描，不是全域实时 PII 检测"；C-04 得到强支持。

## 6. Benchmark Case（供 skill CI 回归）

### BC-01 · "删除路径的 bypass 审计"
- **任务**：在一个大型仓库中，找出所有"删除内容但可能跳过派生索引清理"的调用点，并说明每处的守卫条件与作用域。
- **OpenViking 的正确答案（Gold）**：
  1. `viking_fs.rm` 主路径：先删索引、后删文件（`_ops.py`）；
  2. `fs_service._rm` 的 `no_vector=True` bypass：仅在 `NamespaceProcessingMode.FILE_ONLY`；
  3. `FILE_ONLY` 仅 `viking://system/...`，且该 namespace `supports_reindex=False`；
  4. missing 目标走 `_finish_missing_delete` 清孤儿索引。
- **失败模式**：只读主路径、未发现 bypass → 判为"证据不闭合"；发现 bypass 但未核对 namespace 限定 → 判为"过度告警"。
- **能力维度**：bypass/alternate path 发现 + scope 收敛 + 不把 bypass 直接当缺陷。

### BC-02 · "命名 vs 机制"纠偏
- **任务**：判断 `HierarchicalRetriever` 是否做递归层级遍历。
- **Gold**：否；单次向量召回 + 可选 rerank，层级只体现在返回 tier。
- **失败模式**：按名字推断架构（典型的"文档/命名当实现"错误）。

## 7. 结论

- Package 的**主干结论成立**（删除顺序、检索机制、trusted 鉴权、schema 记忆、渐进披露）。
- 但存在 **1 条 L4 降级、1 条过度概括、3 处遗漏**；其中 `no_vector` bypass 是唯一会改变 KO 边界的遗漏。
- 未发现直接矛盾（CONTRADICTED=0），说明证据链总体可回溯；问题集中在"层级过高"与"bypass 未穷举"，而非事实错误。
- **建议**：Reconciliation 阶段按 §3/§4/§5 修正；Corpus 写入时以修正后版本为准，并保留本报告。
