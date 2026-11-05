# Internal Validation（六维自检）

> 执行者：Synthesizer 自检（非独立审计）。独立 Auditor 盲重建见 `../validation/independent_validation.md`（阶段⑤），本文件不替代它。
> 日期：2026-11-06；范围：`package/` 全部产物 + `repo/` HEAD `5715d3ef3b`。

## 1. Truth（真实性）

抽检 8 条，全部回仓库复核：

| # | 断言 | 复核命令/位置 | 结果 |
|---|---|---|---|
| T-1 | `PauseStrategyHibernate` 在 `pkg/` 生产代码零命中 | `grep -rn PauseStrategyHibernate --include=*.go pkg/ \| grep -v _test.go \| wc -l` = 0 | PASS |
| T-2 | `ensureCheckpointPaused` 仅测试调用 | grep 全仓：定义 `core/util.go:138`，调用仅 `core/util_test.go:1099` | PASS |
| T-3 | `CloudDisk` 代码零命中 | `grep -rn CloudDisk --include=*.go .` = 0；仅 CHANGELOG:38 | PASS |
| T-4 | revision hash legacy 不覆盖多字段 | `revision.go:105-107` 注释明确列出 | PASS（`EK-32` 已按实测重写） |
| T-5 | PreUpgrade 失败不返回 error、继续 Checkpointing | `upgrade_control.go:199-212`：置 reason + event 后 `return nil`，随后 fallthrough | PASS |
| T-6 | `PauseStrategy` 仅被复制、无读取分支 | grep：`revision.go:58/74`、`materialize.go:71`、`clone.go:532/690` 均为赋值 | PASS |
| T-7 | 唤醒置 forever ShutdownTime | `wake.go:150` 使用 `timeout.ForeverReservePausedSandboxDuration` | PASS（数值定义不在本仓库，已删"约 100 年"的推断） |
| T-8 | 暂停前 short requeue 持久化 resume | `auto_pause.go:148-152` + `:394` `resumeScheduleNeedsPersistence` | PASS |

**修正记录**：
1. `EK-32` 原写"revision 复制时不修改 VolumeClaimTemplates 等"，实测为 **legacy hash 不覆盖这些字段**（复制仍在 `:58/:74`）。已重写 statement/evidence/links。
2. EK-44 原写"约 100 年"，常量定义不在本仓库 → 改为"远未来常量，具体值在外部 timeout 包"。
3. `data_model.md` 对应段落同步修正。

## 2. Coverage（覆盖）

- 46 条 EK 覆盖子系统：controller/sandbox（生命周期、auto-pause）、core（checkpoint/upgrade/recycle）、poolautoscaler、sandbox-manager（claim/clone/quota）、sandbox-gateway（wake/jwt/filter）、features、webhook、sandboxset revision、api types。
- **未覆盖（声明）**：`pkg/servers`（31,332 行 E2B 协议）、`pkg/utils`（28,649 行）、`pkg/agent-runtime`（5,247 行）、`pkg/cache`（8,075 行）未逐文件；见 C-07。
- 量化覆盖：按 pkg 行数计，直接检索的模块 ≈ 74,203（controller）+38,333（manager）+7,890（gateway）+9,122（identity 部分）+7,411（webhook）+5,247（agent-runtime 仅入口）= 约 132k / 232k ≈ **57%**（按行数粗估，非精确）。
- 结论：Project Layer 与关键机制覆盖充分；对外 API 契约（servers）存在明确缺口。

## 3. Flow（流一致性）

- 七类流全部产出（`04_flow_atlas/flow-atlas.md`），约 60 条 Edge。
- 抽样核验 symbol 存在性：`Reconcile`、`ensureVolumeClaimTemplates`、`checkTimers`、`calculateStatus`、`handleAutoPause`、`tryResume`、`pauseSandbox`、`EnsureCheckpointForUpgrade`、`performRecreateUpgrade`、`computeDesiredReplicas`、`Wake`、`reconcileLimitedSubject` —— 全部在仓库中定位到。
- State Flow 与 Control Flow 的 phase switch 一一对应；未发现"状态表有、代码无"的转移。
- Flow→KO 交叉校验：
  - KO-01 依赖 C-09/C-10 + E-03/E-04（evidence flow）→ 一致。
  - KO-06 依赖 P-01 + E-03 → 一致。
  - KO-08 依赖 C-01（wake）+ S-04 + P-04 → 一致。
  - 未发现 KO 与 Flow 矛盾。

## 4. Abstraction（抽象）

| 检查 | 结果 |
|---|---|
| 是否存在"单案例→Pattern" | KO-09 单项目且采纳未验证 → 已压到 L3，不升 L4 |
| 是否存在"项目经验→通用 Principle" | 9 条 KO 全部标 `Cross-project validation pending`，无一条写 Law/Principle |
| 是否存在"ADR/文档→实现事实" | C-01/C-02/C-05 均把文档差异标 Observation/Hypothesis，不写成 Fact |
| L4 是否过度 | 8 条 L4 倾向均有一句话稳定关系 + 至少 2 个独立实例；KO-04 的"恢复 patch 应用点"未验证 → 保留限定 |
| 未升维的 EK 是否被丢弃 | 否：9 条未入 KO 的 EK 全部保留在 `02_engineering_knowledge` |

## 5. Counterexample（反例预算）

- 9 条 KO 各 3 个反例攻击 = **27 次**（见 `counterexamples.md`）。
- 其中 **14 次有效削弱**（触发 scope 收紧或降层），完整列表见 `counterexamples.md` 汇总。
- **0 次推翻**（没有 KO 被证伪）。
- 无"0 反例"的 KO；每条均给出搜索证据或代码对照。

## 6. Epistemic（认知状态）

| 状态 | 数量 | 条目 |
|---|---|---|
| Fact（S3/S4） | 46 / 46 | 每条 EK 均有代码位置或测试文件证据 |
| Observation | EK-17（"实现与策略脱节"为观察） | 也在 C-01/C-05 |
| Hypothesis | C-01~C-07 + C-04a（8 条全部） | 无一条进入 KO |
| Validated Pattern | KO-01~KO-08（8 条） | 均单项目，标 cross-project pending |
| Pattern（采纳未验证） | KO-09 | 不升 L4 |

**混淆检查**：
- 无 Hypothesis 被写成 Fact（C-02 的两种可能均未择一）。
- 无单项目 Pattern 被写成 Principle/Law。
- `docs/proposals` 缺失仅写 Observation（C-05），未推断"流程未执行"。

## 结论

- 内部六维：**5 PASS，1 PASS WITH LIMITATION（Coverage）**。
- 修正 3 处（EK-32、EK-44、data_model），均由实测推翻原表述，未弱化证据标准。
- 未执行测试（`go test`/e2e），所有 S4 标注均来自仓库内测试文件的存在与断言内容，不是本轮运行结果——已在 `00_overview.md` 的认知状态声明中说明。
