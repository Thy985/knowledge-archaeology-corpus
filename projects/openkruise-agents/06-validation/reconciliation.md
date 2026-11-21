# Reconciliation（阶段⑤ → 阶段⑥ 修正合并）

> 原则：独立审计不修改原考古产物；修正以"最终口径"落入 corpus。以下每条记录「原命题 → 独立判定 → 最终处理 → 影响面」。

## R-01 · EK-32：revision "不可变字段" → legacy hash 兼容

- **原命题**：模板 revision 复制时不修改 `VolumeClaimTemplates/PersistentContents/Runtimes/Probes`，但复制 `PauseStrategy`。
- **独立证据**：`revision.go:40-84` 的 `buildSandboxTemplateSpec` **复制全部字段**（两处 return 都赋值）；
  `revision.go:105-107` 注释明确 legacy hash 只覆盖 `spec.template`，不包含上述字段。
- **判定**：CONTRADICTED（机制归因错误）。
- **最终处理**：EK-32 重写为"SandboxSet revision hash 保留 legacy 算法以兼容滚动升级"；links 改为
  `mechanism ↔ EK-22`、`constraint → EK-25`；`01-project-layer/data_model.md` 同步。
- **影响面**：EK-32、data_model、KO-02 簇、ek-graph 边类型统计（mechanism 18→19、contrast 23→22）。

## R-02 · EK-44：删除"约 100 年"的数值推断

- **原命题**：唤醒时把 ShutdownTime 设为 `now + ForeverReservePausedSandboxDuration`（约 100 年）。
- **独立证据**：常量在 `pkg/sandbox-gateway/wake/wake.go:150` 被引用，但定义不在本仓库（属外部 `timeout` 包）。
- **判定**：PARTIALLY_CONFIRMED（机制成立，数值无据）。
- **最终处理**：改为"远未来常量，具体值在外部 timeout 包"，不写数值。
- **影响面**：EK-44、KO-08。

## R-03 · EK-23 / KO-04：快照差异链路降为 partial，新增 C-04a

- **原命题**：Checkpoint status 保存 `PodTemplateDelta`，resume 时应用该 patch 重建 Pod（隐含"写入→应用"完整链路）。
- **独立证据**：
  - 定义：`api/v1alpha1/checkpoint_types.go:180-186`；
  - 应用：`pkg/controller/sandbox/core/pod_control.go:170-172` 有生产调用（best-effort，失败 Warning 继续）；
  - 构建：`BuildPodTemplateDelta`（`pkg/controller/checkpoint/utils.go:171`）**无生产调用**，仅 `utils_test.go`；
  - 全仓 grep `PodTemplateDelta` 未发现写 `Status.PodTemplateDelta` 的生产代码。
- **判定**：PARTIALLY_CONFIRMED / DOWNGRADED（读取侧成立，写入侧缺失）。
- **最终处理**：EK-23 statement 加限定；KO-04 标 `Validated Pattern (partial)`；新增 C-04a。
- **影响面**：EK-23、KO-04、Candidates 计数 7→8、README/metadata。

## R-04 · KO-02 簇：5 条 → 4 条

- **原命题**：KO-02（R1 机制簇）包含 EK-17, EK-18, EK-19, EK-27, EK-32。
- **独立证据**：EK-32 主题是"新旧 hash 格式兼容"，不属于"类型声明-实现-可达性"机制；勉强并入会稀释簇内聚性。
- **判定**：DOWNGRADED（聚合归类过宽）。
- **最终处理**：KO-02 簇改为 4 条；matrix / ek-graph / KO 正文同步；未入 KO 的 EK 由 7 条增至 9 条（EK-26、EK-32 加入底座）。
- **影响面**：KO-02、ko-aggregation-matrix、ek-graph、覆盖率统计（37/9≈4.1）。

## R-05 · 行号精度（未修改产物，记录备查）

- **原命题**：SandboxPhase 定义 `sandbox_types.go:504-530`。
- **独立证据**：常量实际为 `511-530`，504 起为类型注释块。
- **判定**：PARTIALLY_CONFIRMED（区间覆盖注释，不影响语义）。
- **处理**：保留原区间（包含类型注释），在验证报告中记录；后续 refresh 可精确到 511-530。

## R-06 · 覆盖率口径（NEEDS_HUMAN_REVIEW）

- **原命题**：覆盖 ~57%。
- **独立证据**：按 pkg 总行数 232,348 与"直接检索模块约 132k"粗估；分母/分子口径未在产物中写全。
- **判定**：NEEDS_HUMAN_REVIEW。
- **处理**：corpus metadata 标 `rough estimate`；不用于任何 KO 论证。

## 未采纳的审计建议

- 未把 `commit`/`securitytokenrefresh` controller 补写成 KO：证据不足且属覆盖缺口（M-02/M-03），留在 Candidates/限制说明。
- 未把 `PodTemplateDelta` 写入侧直接判为"未实现"：可能由外部 runtime 写入，标 Hypothesis（C-04a），不下结论。
