# Reconciliation — ARCH-2026-09-23-001 (E2B)

> 阶段 6 产物。合并 Independent Auditor 的修正到 corpus 顶层文件；原始考古产物保留于 `archaeology-runs/ARCH-2026-09-23-001/package/`（不修改）。

## 修正清单（共 8 项）

| # | 类型 | 对象 | 修正内容 | 证据 |
|---|---|---|---|---|
| R-01 | CONTRADICTED | EK-14 | "RUNTIME_PROBED_PROPS 读取即抛" → **读取返回 stand-in、使用即抛**；补 has trap（`Object.hasOwn` own-keys-only，`name in` 语义） | iam.ts:17-23, :118-149 |
| R-02 | PARTIALLY_CONFIRMED | EK-19 | 证据路径修正：`packages/js-sdk/src/commands/` → `packages/js-sdk/src/sandbox/commands/`；subsystem 归属注明"模块位于 sandbox 子系统内" | 仓库路径实测（find/grep） |
| R-03 | MISSING | EK-02 | 补充：onTimeout 未配置 → autoPause 字段整体省略（API 拥有默认值，与 C-03 呼应） | sandboxApi.ts:1711-1713 |
| R-04 | MISSING | EK-13 | 补充：`SandboxIamTokenType = 'JWT-SVID'`（服务端类型集，"may grow, so any string is allowed"） | sandboxApi.ts:398-399 |
| R-05 | OVER_GENERALIZED | KO-02 | 确认 L4 降级：判别联合为 TS 静态类型特性，跨语言（Python）需运行时校验；保留 L4 但标注语言相关性 | 06_validation.md §1.4 + independent-audit.md §4-3 |
| R-06 | 补充 | EK-11 | `buildNetworkBody` 用 `!= null` 判断（显式 `egressProxy: null` 也视为省略） | sandboxApi.ts:1103 |
| R-07 | 补充 | EK-20 | NotFoundError 本体标 deprecated（FileNotFound/SandboxNotFound 取代，下个大版本移除） | errors.ts:71 |
| R-08 | 路径补全 | Flow Atlas | commands 引用路径补全为 `packages/js-sdk/src/sandbox/commands/index.ts` | 本文件 04_flow-atlas/flows.md |

## 未采纳/保留项

- **C-02 inflight slot 提前释放 TODO**：保留为 Observation + Candidate（代码自陈 TODO，影响未实测）——不因"听起来像问题"升格为缺陷。
- **KO-03/KO-05 等 L4 簇**：Auditor 独立论证通过，无降级。
- **python-sdk 实现体**：覆盖边界声明（C-04），不强制补读。

## Reconciliation 结果

- 顶层 corpus 文件（00_overview.md / 02_engineering-knowledge/ek-graph.md / 04_flow-atlas/flows.md）已写入修正版。
- 原始考古产物完整保留于 `archaeology-runs/ARCH-2026-09-23-001/package/`（含被修正的 EK-14 原文），满足"不覆盖历史 Run / 禁止修改原考古产物"。
- 全部修正可回溯到独立验证报告（06_validation/independent-audit.md）与仓库源码。
