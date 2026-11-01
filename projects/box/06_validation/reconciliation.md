# Reconciliation — ARCH-2026-10-20-001（Strands Box）

> 合并 Independent Validation 修正。只修正 Corpus Artifact，不修改生产 Skill。

## 修正清单

### R-1 · EK-B08 标注文档-实现漂移（修正 E-1）
- **修正内容**: 原文"env:// + aws:// 两 scheme（拒绝其他 scheme at load）"基于 credentials.md → 修正为"设计文档声明两 scheme；实现含五源（env:// / file:// / op:// / aws:// / credsd://，crates/credentials/src/sources/*）；文档与实现漂移已记录，scheme 全集合以 sources/ 为事实基线"
- **证据**: credentials.md "only these two schemes" vs backend.rs:10-29 + sources/{env,file,op,aws,aws_profile,credsd}.rs

### R-2 · EK-B05 补 judge/finding 细节（修正 E-2）
- **修正内容**: temporal 规则节补"加载期 judge 做跨 permit 覆盖率判定：无条件 permit 使 temporal permit 惰性→`InertTemporalPermit` 拒绝加载；heuristic 形状 warn"（cap.rs:22-74）
- **证据**: policy/cap.rs

### R-3 · EK-B12 补"统一路径拼写防 forbid 逃逸"（修正 OVER_GENERALIZED）
- **修正内容**: 除"解释器暴露面最宽"外，补：两解释器必须报同一路径拼写（home 缩写统一），否则同 `context.input.path` 对 permit fail-closed、对 **forbid fail-open**——拼写统一是 forbid 防逃逸机制
- **证据**: policy/adapters/script.rs:32-40

### R-4 · 新增 EK-B21 · box 自身失败与 workload 失败分离（采纳 S-2）
- **新增 EK**: workload 非零退出仍 `Ok`（box 做了它的事）；只有 `BoxError` 是 box 自身失败；teardown 记录 BoxStopped/refused 后 `drained()` 唯一出口（run.rs:91-103）
- **links**: subsystem→EK-B20（生命周期）；dependency→EK-B19

### R-5 · 新增 EK-B22 · 遏制测试 QUARANTINE 治理（修正 E-3）
- **新增 EK**: test-integ/QUARANTINE.md——编译期精确 allowlist `TEMPORARY_QUARANTINE` + 单元测试钉住列表 + SKIP 只接受恰一条授权注释；未列出 cell 或错误注释=integrity 失败
- **links**: contrast↔EK-B20（加载期拒绝同类"防静默失效"）；subsystem→test-integ

### R-6 · C-B01 标注 NEEDS_HUMAN_REVIEW（维持）
- **修正内容**: C-B01（组织级多 box 凭据隔离）标注 NEEDS_HUMAN_REVIEW（引擎侧 box 粒度边界明确；跨 box 编排属 operator 侧，未见原生机制）

## 未修改项
- KO-B01~B05（无攻击成立）；Flow Atlas（七类流 Edge 全部确认）；00/01/03/04/05 主体
- 原考古产物不可修改原则：以上修正以 corpus 落盘版本为准，本地 package 保留初版 + 本文件记录差异

## Reconciliation 后 EK 统计
- EK 总数: 22（B01~B22）；links 覆盖率 100%；平均出边 ~1.7；游离 0%
- 6 项修正全部可反向追溯独立证据（见 independent_validation_report.md）
