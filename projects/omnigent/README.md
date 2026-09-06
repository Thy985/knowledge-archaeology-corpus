# Omnigent — Project Archaeology Index

> **一句话定位**：开源 meta-harness（Databricks，Apache 2.0，alpha）——在 Claude Code / Codex / Cursor / OpenCode / Hermes / Pi 等 11 个 agent 之上提供统一编排层：组合、控制（策略/沙箱/花销）、协作（实时共享会话）。

## 考古时间线

| Run | 日期 | Commit | 版本 | 结果 |
|-----|------|--------|------|------|
| [ARCH-2026-09-07-001](archaeology-runs/ARCH-2026-09-07-001/) | 2026-09-07 | 381bf638fb31e6a51990d9dab54ea9ef4b933711 | 0.13.0.dev0（release v0.12.0） | ✅ 完成（30 EK / 7 KO / 4 CM / 4 M / 8 Candidates / 独立验证 26 CONFIRMED） |

## 产物清单

| 层 | 文件 |
|----|------|
| Overview | [00_overview.md](00_overview.md) |
| Project Layer | [01_project-layer.md](01_project-layer.md)（[detail](01_project-layer/project-layer.md)） |
| Engineering Knowledge（EK Graph） | [02_engineering-knowledge.md](02_engineering-knowledge.md)（[detail](02_engineering-knowledge/ek-graph.md)） |
| Knowledge Layer（KO/CM/M） | [03_knowledge-layer.md](03_knowledge-layer.md)（[detail](03_knowledge-layer/knowledge-layer.md)） |
| Flow Atlas（七类流） | [04_flow-atlas.md](04_flow-atlas.md)（[detail](04_flow-atlas/flow-atlas.md)） |
| Candidates | [05_candidates.md](05_candidates.md)（[detail](05_candidates/candidates.md)） |
| Validation & Reconciliation | [06_validation.md](06_validation.md)（[detail](06_validation/validation.md)） |
| 元数据 | [metadata.yaml](metadata.yaml) |
| 原始 Run 快照 | [archaeology-runs/ARCH-2026-09-07-001/](archaeology-runs/ARCH-2026-09-07-001/) |

## 关键结论（速览）

1. **策略执行点跟随控制权**：runner 拥有 MCP dispatch → function 型 policy 下移 runner（runner/policy.py），label/prompt 型按能力边界留 server——执行点跟着控制权走。
2. **人机闸门超时语义是产品决策**：ASK 默认等待 86400s（旧 120s 静默拒绝被认定为 bug）；headless 显式 fail-closed。
3. **身份持久 vs 资源易逝**：host_id 绑定存活、sandbox generation 可换；用户凭据从不进沙箱（CredentialProxy swap-on-access + `oa_cred_*` 占位符 + 跨 host 403 守卫）。
4. **失败可解释性是投资**：wedged LLM→connectivity 单槽健康记录（#1119）、崩溃三 chokepoint、OBSERVABILITY 诚实审计。

## 与其它 Corpus 项目的连接

- **deepseek-harness / rampart**：审批-权限-沙箱主题（KO-03/KO-06 同域）
- **evolver**：自愈模式族（KO-05 同域：自我更新可恢复）
- **dsh-memory-evolve**：凭据/锁/写入安全（CredentialProxy 与跨设备同步锁同域）
- **skillfortify / memgraphrag**：治理闭环（Policy Flow）与观测投资

## 验证状态

- Truth / Flow / Abstraction / Counterexample / Epistemic 全 PASS（详见 06）
- 覆盖缺口（诚实声明）：测试未运行（环境缺重型依赖，S3 证据）、巨型文件仅头部精读、web/desktop 未覆盖
- 独立 Auditor 发现并合入：EK-31 CredentialProxy（MISSING 修复）、04.5 URL 修正、EK-29 时差补注

---
*Corpus entry · knowledge-archaeology v3.2 · 2026-09-07*
