# 05 · Candidates（未验证假设 / 跨项目候选 / 待人工审查）

> 纪律：Candidate ≠ Knowledge。每条必须写清"当前证据 / 缺失证据 / 验证路径"。
> 禁止把 Candidate 的结论写进 KO 或 EK 的 claim。

| ID | 类型 | 命题 | 当前证据 | 缺失证据 | 验证路径 | 风险 |
|---|---|---|---|---|---|---|
| **C-01** | Hypothesis（可证伪） | `add_resource` 覆盖同名资源时可能先清旧内容再写新内容，违反 KO-02 的"写成功才删" | `09-transaction.md` 描述"清理解析产物"，但未明确是否删除既有内容；`ContextUpdatePlan` 代码未逐行读 | `plan` 编译阶段的 content actions 是否含 delete；overwrite 分支的失败处理 | 读 `resource_service.py` + `ContextUpdatePlan` 实现；写并发覆盖失败注入测试 | 若成立：KO-02 的 scope 需收窄为"memory 写入路径" |
| **C-02** | Hypothesis | Phase 1 marker 与队列消费是幂等的，重复消费不会产生重复记忆 | 有 archive number 单调分配 + version 递增；未见幂等键字段 | marker 唯一键；重复消费的 diff 去重逻辑 | 读 `session_commit_processor` 的 task 去重；构造重复 task 测试 | 若不幂等：KO-05 的"发布点"结论需加"至少一次消费"限定 |
| **C-03** | Observation（待验证） | ACL 的"短暂多版本"窗口在真实并发下可能持续数秒，而非"瞬时" | `15-acl.md` 明写不保证强一致；`acl_mode IN [inherit,restricted]` 过滤 | 索引更新延迟分布；重索引队列积压时的窗口 | 运行时压测：并发改 ACL + 检索，测不一致持续时长 | 权限泄漏窗口被低估 |
| **C-04** | Hypothesis（安全） | 隐私机制只覆盖"已注册值"（`.meta.json` 中有记录的 value），未注册的自由文本敏感信息仍会进入正文/索引 | `13-privacy.md` 描述"registered-value protection"；`PrivacySanitizer` 依赖 `PrivacyFieldReference` | 扫描器是否对未注册模式做通用 PII 检测 | 读 `privacy/scan.py` + `rules.py`；构造未注册 API key 文本测试 | 过度信任隐私模块 |
| **C-05** | Cross-project Candidate | "索引不领先于源"可能在所有 read-model 系统成立 | OpenViking 5 条 EK 支撑；corpus 中 mem0/graphiti 未显式声明 | 其他 2+ 项目的删除/更新顺序 | 对 mem0、graphiti、letta 做同类考古对照 | 单项目升维 |
| **C-06** | Hypothesis | `config/merge.py` 的账户级覆盖是"深合并"，嵌套 dict 中 list 的合并策略可能整段替换 | `merge.py` 存在；未逐行读 | list/dict 冲突时的实际行为 | 读 `merge.py` + 单测 `tests/config/` | 配置误覆盖 |
| **C-07** | Needs human review | `crates/ragfs` 的 PathLockManager 是否真能保证 E/T/MV 语义，Python 层无法验证 | 文档 `09-transaction.md` 详尽；Python 侧只调用 binding | Rust 实现的原子性、崩溃恢复、token 过期 | 读 Rust 源码（130k 行中定位 lock 模块）+ Rust 测试 | 锁语义仅纸面成立 |
| **C-08** | Needs human review | `X-OpenViking-Actor-Peer` 允许调用方声明"我是某个 peer"，若上游不可信，可越权读他人 peer 记忆 | `auth/__init__.py` 接收该 header；`retrieval_targets` 用它扩展目录 | 是否校验 actor_peer 与 user 的关系；是否仅 trusted 模式允许 | 读 `normalize_actor_peer_header` + peer 归属校验 | 横向越权 |
| **C-09** | Observation（运维） | 向量记录新格式"降级只能回滚到首次新格式写入前的备份"——一旦写入，旧版本不可读新记录 | `05-storage.md` 明写 | 混合版本部署的实际行为 | 构造新旧版本混跑测试 | 升级后回滚失败 |
| **C-10** | Hypothesis | `experiences` 三段式蒸馏的实际质量（可执行性、抽象度）未知，prompt 约束不等于模型遵守 | YAML prompt 极详细（Situation/Approach/Reflect、≤8 bullets） | 真实抽取样本与人工评分 | 跑 benchmark 或抽样 20 条经验人工评估 | 把"设计意图"当"运行效果" |
| **C-11** | Needs human review（法律） | AGPL-3.0 下以 OpenViking 提供网络服务可能触发源码提供义务 | `LICENSE` + `pyproject.toml:19` | 具体部署形态（是否修改、是否分发） | 法务评估；非技术结论 | 合规风险 |
| **C-12** | Observation | "HierarchicalRetriever" 命名暗示递归树遍历，实际是单次向量召回；调用方可能误判召回机制 | `hierarchical_retriever.py:102-240` | 是否有其他代码路径做真正递归 | 全局搜索 `Hierarchical` 调用点；读 README 表述 | 集成方预期错配 |

## 与 KO 的关系

| Candidate | 影响的 KO | 处理方式 |
|---|---|---|
| C-01 | KO-02 | KO-02 已在反例 3 中显式标注"未验证"；不修改结论 |
| C-02 | KO-05 | KO-05 Epistemic 标 `partial: idempotency unverified` |
| C-03 | KO-06 | 不改变结论（文档已承认为最终一致） |
| C-04 | KO-06 | 不改变结论；作为隐私模块的边界 |
| C-05 | KO-01 | 保持 `Cross-project validation pending` |
| C-12 | KO-08 | 作为"隐喻≠实现"的补充证据 |
