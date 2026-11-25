# 06 · Reconciliation — Package 修正记录

> 只修正 Package Artifact，不修改生产 Skill、不修改目标仓库。所有修正在 `03-knowledge-layer` 中版本化，保留原 Validation 判定。

## R-01 · KO-02 scope 收紧（PARTIALLY_CONFIRMED）

- 原文：`任何同机进程都可以伪造 Host 并直接 POST /call`
- 修正：`一个能建立 loopback 连接并可设置 Host 的本地调用方`
- 原因：代码证明无 auth + 任意 roots + 无审计；不证明任意 UID / 任意网络命名空间可达。多 UID、容器、防火墙、代理均未测。
- 影响：KO-02 保留 L4；C-01/C-06 仍为 hypothesis/needs review。Corpus 不把它写成“所有本地部署必然可被利用”。

## R-02 · KO-03 降为 L3（DOWNGRADED）

- 原文：`abstraction: L4`，claim 宏观到“任何工具启动/attach 资源”。
- 修正：`abstraction: L3`，claim 限定“本项目的两个隔离面（session 与 keychain）”。
- 原因：只有两个实例，且 session 隔离 bridge、keychain 隔离浏览器 flag，不是同一子系统的两个独立实现；L4 认知模型过强。
- 影响：KO-03 仍是有效项目级 Pattern，但不能跨项目推广。

## R-03 · KO-05 显式 legacy 例外（PARTIALLY_CONFIRMED）

- 原文：`ref 属于当前 epoch，且零突变`，容易被读成“所有 ref 都绑定 epoch”。
- 修正：claim 增加“没有 `gN:` 前缀的 legacy ref 不绑定原始 epoch，只受当前页面状态约束”。
- 原因：`checkUidGeneration('@5', 7)` 返回 `stale:false`（`test/snapshot.test.ts:200-240`），但 `parseUidFresh` 仍过页面状态门。
- 影响：C-04 保留为 uncertain_pattern；不把 legacy 兼容说成完整安全边界。

## R-04 · KO-08 scope 收紧（OVER_GENERALIZED）

- 原文：`质量与发布声明不能只靠约定`，隐含跨项目原则。
- 修正：`在本仓库的 release/CI 治理中`，并写明“不跨项目推广”。
- 原因：只有单一仓库证据；结论正确但 novelty 低，且 L3 Pattern 不足以支撑“所有仓库都必须”。
- 影响：KO-08 保留 L3，但 Validation 标 OVER_GENERALIZED；不作为跨项目 Principle。

## R-05 · KO-01 scope 收紧（CONFIRMED 项目级）

- 修正：scope 增加“同一 bridge + 同一 session”，并注明 deep health 假阳性未排除。
- 原因：attach 到同一外部浏览器的多 session 会 cross-talk（EK-10），三份连续性不能覆盖浏览器层共享状态。

## R-06 · 证据索引修正

- `EV-030` 跳号，不补造；原因是编号沿用历史 run 的 evidence 区间，避免与已有 Corpus 冲突。
- `EV-046` 的 status 为 `runtime`，只代表本机 `pnpm test`；不代表 CI Node 24 或 live Chrome。
- 所有 49 行 source 均回到仓库文件/行号或 commit；没有外部网页、没有记忆。

## R-07 · 未接受的修正

- 不把 `isBridgeProcess` 子串匹配升级为“已验证漏洞”（C-02 保持 hypothesis）。
- 不把“同机任意用户可达”写成 Fact（C-01 保持 hypothesis）。
- 不把 `roots` 的 live 文件写范围写成 Fact（C-06 保持 hypothesis）。
- 不因 KO 形成而删除任何 EK；36 条 EK 全部保留。
- 不把 Validation 的 `PARTIAL` 改成 `PASS`；无 live Chrome、无运行时攻击、无跨 UID。

## 校验清单

- [x] EK 36 / KO 8 / Flow 48 / Evidence 49 / Candidates 12
- [x] 每个 KO 有 `aggregation_rule`，`derivation.facts` 等于 `cluster_eks`
- [x] 每个 KO 有 Flow Traceability（至少 1 条 L1 fact → edge）
- [x] 每个 KO 有 counterexample / scope / validation
- [x] 无 L5、无 Law、无跨项目 Principle
- [x] 无 `KO-xx` 出现在 EK `links`（links 只指向 EK）
- [x] 无孤立 EK（36/36 有 links）
- [x] 所有 Evidence 可回到仓库文件/行号或 commit
- [x] `run_metadata.yaml` 记录准确 commit / skill version / metrics / validation result
- [x] Corpus 不覆盖历史 Run；新目录 `projects/chrome-devtools-axi/`
