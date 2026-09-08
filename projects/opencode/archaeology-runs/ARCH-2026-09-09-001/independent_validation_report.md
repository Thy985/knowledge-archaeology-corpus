# Independent Validation Report — OpenCode（ARCH-2026-09-09-001）

> Auditor 盲重建流程：独立重读仓库（不读考古产物）建立 Independent Findings → 再对比考古产物 → 输出判定。考古产物未被修改。

## 判定统计

| 判定 | 数量 |
|------|------|
| CONFIRMED | 22 |
| PARTIALLY_CONFIRMED | 3 |
| DOWNGRADED | 0 |
| OVER_GENERALIZED | 0 |
| MISSING | 2 |
| CONTRADICTED | 0 |
| NEEDS_HUMAN_REVIEW | 1 |

（MISSING 两项为审计检查项，已收纳进考古产物 05 Candidates：C-02 挂起失败模式、C-04 候选卡失真根因；非遗漏性错误。）

## 原考古最重要的 3 个成功

1. **候选卡事实失真被一手修正**：盲重建独立调用 GitHub API（archived: True / 13,727★ / 2025-09-17 归档）并核对仓库结构（Go 单体 TUI、无 TS/Hono）——考古产物以仓库实际为准构建知识，把"147k★/headless HTTP"失真记录为事实修正而非吸收进结论。这是 Source Fidelity 的教科书案例。
2. **真实 S4 测试证据**：Auditor 独立跑通 `go build` + `go test`（3 包 ok / tools FAIL panic），panic 堆栈（config.go:874 ← ls.go:99 ← ls_test.go:139）精确可回溯——测试揭示 WorkingDirectory 无保护全局状态 bug，优于上轮 Dogwood（S4=0）。
3. **审批模型三层逻辑准确捕获**：autoApproveSessions → sessionPermissions 缓存 → pendingRequests 同步阻塞，与代码（permission.go:74-112）完全一致；"无超时阻塞 vs omnigent ASK 86400s 超时"对照判断恰当（未越级断言优劣）。

## 最重要的 3 个错误

1. **（弱错误）EK-15/KO-06 含推断性综合**："增长与成熟度解耦"是 Pattern 级判断（从 4 测试文件 + 归档推出），已归 KO-06/CM-04 并标 Cross-project pending——但 EK-15 的表述略超纯事实（"早期项目"是解释性词汇）。影响：低，不改变结论。
2. **（弱错误）测试画像非全量**：go test 仅覆盖 4 个测试文件所在包；全量 `go test ./...` 在依赖下载阶段超时（exit 124）——考古产物已如实标注"3 ok + 1 FAIL"，未宣称全量通过。影响：S4 证据覆盖限于 4 包。
3. **（弱错误）C-04 未闭合**：候选卡 headless 描述的来源（crush 混淆 vs 数据错误 vs 未公开分支）未定论——已标 NEEDS_HUMAN_REVIEW。影响：仅影响 KnowlegeMap 数据质量建议，不影响 opencode 考古本体。

## 关键遗漏检查

- **是否存在关键遗漏？** 无结构级遗漏。Auditor 独立发现项（权限三层逻辑、monkey patch、panic 堆栈、SQLC/viper 注入）全部已被考古产物覆盖。
- **是否存在错误升维？** 无。6 KO 全标 Cross-project validation pending（KO-01 三项目弱印证 n=3 如实标注）；4 CM 有代码证据 + 对照；3 M 标"OpenCode 验证"。
- **是否存在事实错误？** 未发现。抽查 symbol/行号/API 数据（13,727★、archived、2025-09-17）全部真实。
- **是否存在 Flow 错误？** 未发现。七类流 Edge 全部可回溯（agent.go:198/276/352、permission.go:74/110、bash.go:270、config.go:874）。

## 攻击结果（bypass/override/exception/alternate/direct call/admin/fallback/legacy）

| 攻击面 | 结果 | 覆盖 |
|--------|------|------|
| bypass（AutoApproveSession） | 存在：会话级信任提升 | EK-02 ✅ |
| override（sessionPermissions 缓存） | 存在：同参重复批准跳过 | EK-02 ✅ |
| exception（权限拒绝） | ErrorPermissionDenied | EK-01 ✅ |
| alternate path（MCP vs 内置工具） | 并存 + 前缀命名 | EK-03/06 ✅ |
| direct call（agent.Run 唯一入口） | 确认 | EK-04 ✅ |
| fallback | 无 | — |
| legacy（归档迁移） | 2025-09-17 → crush | EK-18 ✅ |

## 新 Benchmark / Regression Case（3 个，留作 skill 候选）

1. **候选卡一手事实核对**：考古前强制 GitHub API 复核 star/archived/架构——本 run 候选卡失真 10 倍（147k vs 13.7k），防"基于失真画像构建知识"。
2. **GOTOOLCHAIN 自动下载路径**：环境工具链版本不匹配时先试自动下载（Go GOTOOLCHAIN=auto），而非直接放弃——本 run 借此获得 S4 证据。
3. **panic-instead-of-error 检测**：全局状态（WorkingDirectory）无保护是早期 harness 常见失败模式，可作"失败路径覆盖"检查基准。

## 结论

考古产物通过独立验证（22 CONFIRMED / 3 PARTIAL / 0 错误升维 / 0 事实错误）。无 Reconciliation 补录修正；C-02/C-04 保留为 Candidates。亮点：真实 S4 测试证据（含 1 个 FAIL 揭示 bug）；局限：测试覆盖限于 4 包，C-04 需人工审查 crush 架构后闭合。
