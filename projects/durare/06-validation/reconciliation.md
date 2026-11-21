# Reconciliation — ARCH-2026-11-04-001（durare）

> 阶段⑥：合并阶段⑤独立 Auditor 的盲重建修正意见到 corpus 终稿。修正 Corpus Artifact 而非生产 Skill。原考古产物（run 目录 package）同步更新。

## 合并修正清单（3 项）

| # | 修正 | 来源判定 | 落点 |
|---|---|---|---|
| 1 | **KO-04 表述修正（DOWNGRADED 处理）**：原 claim "不靠事务序列化、靠单条原子语句"过度——盲重建发现 `insert_child_workflow` 用显式事务 + `SELECT ... FOR UPDATE` 行锁 + 重放检查（F-7）。修正为"仲裁决定权在数据库而非应用锁，形式因操作而异：单语句 CAS / ON CONFLICT / CTE / 事务+行锁原子原语族"；簇内 EK 由 4→5（新增 EK-52）。 | PARTIALLY_CONFIRMED → 表述收紧（非推翻） | 03 层 KO-04 + 02 层新增 EK-52 + KO 矩阵 |
| 2 | **新增 EK-51（MISSING-1）**：`observation_error()`（execution.rs L140-150）——无本地 execution latch 的观察者（Client 直连 provider）把 storage failure 转 ObservationFailed 的控制错误通道；与 body 内 `provider_error()` 形成"有 latch/无 latch"两条分类路径。 | MISSING → 补充 | 02 层 + KO-03 簇（6→7 EK） |
| 3 | **新增 EK-52（MISSING-2）**：`insert_child_workflow`（postgres.rs）事务+FOR UPDATE 行锁仲裁 + 重放检查 + CANCELLED 检查；child 插入是"DB 仲裁"主题下的事务路径，与单语句 CAS 并存。 | MISSING → 补充 | 02 层 + KO-04 簇（4→5 EK） |

## 管道顺序 / 范围收紧确认

- **未推翻任何 KO**：10 个 KO 全部保留；KO-04 表述修正不影响成立性。
- **证据等级**：EK-51/52 均为 S3（实现存在，注释阐明意图）；无 S5 升级（无运行时环境）。
- **判定统计（终版）**：CONFIRMED 22 / PARTIALLY_CONFIRMED 2 / DOWNGRADED 1 / OVER_GENERALIZED 0 / MISSING 2（已补）/ CONTRADICTED 0 / NEEDS_HUMAN_REVIEW 1（C-02）。
- **质量指标（终版）**：EK 52（全部有 links，平均出边 ≈2.15，游离 0）；KO 10（聚合规则覆盖 100%，平均簇规模 5.0）；Candidates 6；Flow 七类 23 edge。
- **Skill Evolution**：未触发（3 处修正均属本项目考古 scope 收紧，无系统性 skill 缺陷证据）。

## 写入前检查（六项）

- [x] git diff：仅新增 projects/durare/，无改动其他项目/仓库文件
- [x] schema check：metadata.yaml 字段与 substrate 模板一致（project/slug/repository/version/commit/commit_message/commit_timestamp/skill_version/archaeologized_at/mode/run_id/source_entry/summary）
- [x] duplicate check：projects/durare/ 为新顶级目录，corpus 无同名项目；archaeology-runs/ARCH-2026-11-04-001 唯一
- [x] evidence reference check：EK/KO 证据引用均可回溯到 commit `87ed2db` 的 repo 文件:符号；S4 标注统一"测试存在未经本环境运行"
- [x] metadata check：run_id/commit/skill_version 与 package/run_metadata.yaml 一致
- [x] structure check：目录结构 00-06 + archaeology-runs + README + metadata.yaml，与 restate/substrate 模板对齐

## 遗留（不阻塞交付）

- C-02（AdminServer 无认证 by design）标记 NEEDS_HUMAN_REVIEW，待 Owner 评估。
- open PR #58~#63 等待合入（勿重放）。
