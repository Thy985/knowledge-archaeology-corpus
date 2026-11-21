# 06 · Reconciliation（独立验证 → Package 合并）

> 原则：**修正 Corpus Artifact，不修改生产 Skill**。独立 Auditor 的报告原样保留（`independent-validation.md`），本文件只记录"采纳了什么 / 不采纳什么 / 为什么"。

## 1. 采纳的修正（7 项）

| # | 对象 | 修正前 | 修正后 | 依据 | 影响 |
|---|---|---|---|---|---|
| R-01 | KO-02 | L4 认知模型 | **L3 工程模式** | IF-08/IF-09：仅 memory 写入路径有完整代码证据，mv/add_resource 为 doc-only 或未验证 | 知识层级下调；L4 版本移入 C-05 |
| R-02 | KO-01 | 未声明 scope | 增加 **scope: indexed namespaces only**（`viking://system` 为 FILE_ONLY） | IF-02/IF-03 | 不变量边界明确 |
| R-03 | KO-07 | L3 Pattern（含张力） | **Disputed / 不用于跨项目推广** | IF-04 + 2 条真反例 | 防止过度迁移 |
| R-04 | EK-14 | 六类删除守卫 | **七类**（新增批级 unresolved upsert 守卫） | IF-04 | 守卫完整性 |
| R-05 | EK-16 | 仅条级 `resolution_skip` | **批级 + 条级两层** | IF-04 | 机制描述准确 |
| R-06 | Flow Atlas | 74 Edge / 59 code | **76 Edge / 61 code**；新增 FC-23（`no_vector` bypass）、FA-09（预检后 `bypass_acl`） | IF-02/IF-10 | 覆盖 bypass / alternate path |
| R-07 | Candidates | C-04 缺证据 | 补 **privacy scan 硬限流**（1000 节点/1MB/16MB/60s/1 并发/20 files/s），强化"非实时通用 PII 检测"判断 | IF-11 | 候选证据升级 |

## 2. 未采纳 / 未变更

| 对象 | Auditor 建议 | 处理 | 理由 |
|---|---|---|---|
| KO-01 L4 | 未质疑层级 | 保留 L4 + cross-project pending | 5 条 EK 跨 4 条路径汇聚，且修正后 scope 更清晰 |
| KO-06 L4 | 未质疑 | 保留 L4 | 5 条 EK 跨 auth/ACL/config；结构论证成立 |
| KO-03/04/05/08 | 未质疑 | 保留 L3 | 单项目证据，未越级 |
| C-01/C-02 | 未要求立即验证 | 保留 Candidate | 本轮只读、不做运行时注入；不伪造结论 |
| C-07（Rust 锁） | 建议读 Rust | **未执行** | 本轮 scope 明确排除 `crates/`；如实标 Rust-unread，不假装验证 |
| C-11（AGPL） | — | 保留 Needs human review | 法律结论超出技术考古 scope |

## 3. 修正后的质量指标

| 指标 | 修正前 | 修正后 |
|---|---:|---:|
| EK 数 | 36 | 36 |
| EK 边数 | 98 | 98（仅描述修正，无新增边） |
| Flow Edge | 74 | 76 |
| flow code 支撑 | 59 | 61 |
| KO 数 | 8 | 8 |
| L4 KO | 3 | **2**（KO-01、KO-06） |
| L3 KO | 5 | **6**（KO-02 降级、KO-07 标 disputed） |
| 真反例 | 2 | 2 |

## 4. 冲突裁决记录

- **"删除顺序" vs "no_vector bypass"**：表面冲突，实为 scope 不同。裁决：不变量保留，增加 namespace scope；bypass 作为 Flow Edge 记录，不作为缺陷。
- **"写乐观读保守" vs 两处静默失败**：裁决为 **Disputed**，不强行统一。保留原始观察，但取消其模式推广资格。
- **"文档说 mv 复制→更新→删源" vs 代码未见**：裁决为 doc-only，不计入 Fact；不因文档详尽而升 S3。

## 5. 结论

Reconciliation 完成，Package 以修正后版本进入 Corpus。所有修正均可在 `independent-validation.md` 找到对应 IF 编号；未修改生产 Skill，未触发 Skill Improvement Proposal（3 处问题属执行纪律，非系统性缺陷）。
