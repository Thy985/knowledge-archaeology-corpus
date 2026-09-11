# 06 · Validation & Evidence — OpenCode

> 独立 Auditor 盲重建流程：先独立重读仓库（不读考古产物）建立 Independent Findings，再对比。本文件记录验证结论与质量指标。

## 06.0 验证方法说明

- **运行环境**：环境有 go 1.23 + GOTOOLCHAIN=auto（自动下载 go1.24.0 成功）；`go build ./...` 通过；`go test` 部分可运行（依赖下载完成后）。
- **测试运行画像（本 run 实测，S4 级）**：
  - ✅ ok：internal/llm/prompt（0.125s）
  - ✅ ok：internal/tui/theme（0.016s）
  - ✅ ok：internal/tui/components/dialog（0.103s）
  - ❌ FAIL：internal/llm/tools（0.053s）——ls_test.go:139 → ls.go:99 → **config.WorkingDirectory() panic（config.go:874）**
- **盲重建**：Auditor 独立重读仓库（README/go.mod/结构/permission/agent/tools/lsp/db），实测编译与测试，再与考古产物比对。

## 06.1 判定统计（独立 Auditor）

| 判定 | 数量 | 对象 |
|------|------|------|
| CONFIRMED | 22 | EK-01~13, 16~18, 21~27（代码/结构直接支持） |
| PARTIALLY_CONFIRMED | 3 | EK-15（测试低覆盖确认，但"增长 vs 成熟"判断为推断）、EK-19（star 数确认，但失真根因未定）、EK-20（测试运行确认，但非全量） |
| DOWNGRADED | 0 | — |
| OVER_GENERALIZED | 0 | — |
| MISSING | 2 | 见 06.3 |
| CONTRADICTED | 0 | — |
| NEEDS_HUMAN_REVIEW | 1 | C-04（候选卡失真根因：crush 混淆 vs 数据错误） |

## 06.2 独立 Auditor 的成功发现（Top 3）

1. **候选卡事实失真被捕获并修正**：盲重建独立验证 GitHub API（archived: True / 13,727★ / 2025-09-17）与仓库结构（Go 单体 TUI、无 TS/Hono）——考古产物 00/05 完整记录偏差，未基于失真画像构建知识（Source Fidelity 成功）。
2. **S4 测试证据真实获得**：编译通过 + 4 包测试实测（3 ok / 1 FAIL panic）——比 Dogwood 轮（无工具链）证据更强；panic 堆栈（config.go:874 ← ls.go:99 ← ls_test.go:139）精确可回溯。
3. **审批模型细节被准确捕获**：同步阻塞（无超时）、autoApproveSessions、sessionPermissions 缓存三层逻辑与代码完全一致（permission.go:74-112），且与 omnigent ASK 的对照判断恰当（未越级断言）。

## 06.3 关键错误 / 遗漏 / 攻击结果

### 考古产物 3 个最弱处（Auditor 视角）
1. **EK-15/06 的"增长 vs 成熟"为推断性综合**：测试覆盖低是事实，但"生态信号与工程质量正交"是 Pattern 级判断（已归 KO-06/CM-04 并标注 Cross-project pending）——不构成错误，但需明确。
2. **C-04 未闭合**：候选卡 headless 描述的来源未定（crush 混淆 vs 数据错误）——需人工审查 + 后续核实 crush。
3. **测试非全量**：go test 仅覆盖 4 个测试文件所在包；其余 22 个包无测试文件（"no test files"），非"全部通过"——已如实标注。

### 攻击项
- **单案例→Pattern**：KO-01~06 全部标 Cross-project validation pending（KO-01 已有 3 项目弱印证，如实标注 n=3）✅
- **Pattern→L4**：4 CM 均有代码证据 + 对照（omnigent/Dogwood/DeepSeek Harness），未写成 Law ✅
- **项目经验→通用 Principle**：M-01~03 标"OpenCode 验证" ✅
- **ADR→实现事实**：Cargo 无；README Early Development/归档声明为自述（S2），未当实现行为断言 ✅
- **Flow Edge 真实性**：07 类流全部标注 symbol/file；抽查 agent.go:198/276/352、permission.go:74/110、bash.go:270、config.go:874 均真实存在 ✅
- **bypass/override/exception/alternate/direct call**：
  - bypass：AutoApproveSession = 权限门显式绕过路径 ✅ 已覆盖（EK-02）
  - override：sessionPermissions 缓存命中 = 重复批准跳过 ✅ 已覆盖
  - exception：ErrorPermissionDenied ✅ 已覆盖
  - alternate path：MCP 工具与内置工具并存 ✅ 已覆盖
  - direct call：agent.Run 为唯一入口 ✅ 已覆盖
  - fallback：无；legacy：归档状态（迁移 crush）✅ 已覆盖
- **Epistemic 状态**：Fact（L1）↔ EK（L2）↔ KO（L3）↔ CM（L4）↔ M（L5）分层无混淆；Hypothesis 全部进 05 ✅

## 06.4 关键遗漏检查

- **是否遗漏重要 Engineering Knowledge？** 无结构级遗漏。Auditor 独立发现项（权限三层逻辑、monkey patch、panic 堆栈、SQLC 持久化）均已覆盖。
- **是否错误升维？** 无。KO/CM 均带回溯与状态标注。
- **是否存在事实错误？** 未发现。抽查 symbol/行号/API 数据全部真实。
- **是否存在 Flow 错误？** 未发现。
- **是否发现新的 Benchmark / Regression Case？** 3 个：
  1. **候选卡一手事实核对**：考古前必须 GitHub API 复核 star/archived/架构——本 run 候选卡失真 10 倍，可作 benchmark 强制项。
  2. **测试运行可行性判定**：环境工具链缺失时先试 GOTOOLCHAIN=auto（Go）等自动下载路径，而非直接放弃——本 run 获得 S4 证据优于 Dogwood 轮。
  3. **panic-instead-of-error 检测**：全局状态无保护（WorkingDirectory）是早期 harness 常见缺陷，可作"失败路径覆盖"检查基准。

## 06.5 质量指标

| 指标 | 值 |
|------|-----|
| EK 总数 | 27 |
| EK 有 links | 27（100%） |
| 游离 EK | 0（<20% ✅） |
| 平均出边 | ~2.0（≥1 ✅） |
| KO 聚合规则覆盖率 | 100%（6/6） |
| KO 平均簇规模 | 4.5（3~12 ✅） |
| 假聚合（同子系统=理由） | 0 |
| KO/CM/M | 6/4/3 |
| Candidates | 8 |
| 证据等级分布 | S3×20 + S4×3 + S4F×2 + S2×2 |
| 测试运行 | 编译 ✅；4 测试包：3 ok / 1 FAIL（panic） |
| 交叉校验（Flow→KO） | 6/6 通过 |
| 单案例→Pattern 越级 | 0 |
| Hypothesis 冒充 Fact | 0 |

## 06.6 Reconciliation 记录

- Auditor 独立发现与考古产物一致，无补录修正。
- 保留项：C-04（候选卡失真根因）、C-02（挂起失败模式）→ 待人工审查 / 后续核对 crush。

## 06.7 验证结论

**考古产物通过验证**（Truth / Coverage / Causality / Flow / Abstraction / Counterexample / Epistemic 七项均通过，无 CONTRADICTED；2 MISSING 为审计项已收纳于 05）。**亮点**：本轮获得真实 S4 测试证据（编译 + 3 包通过 + 1 包 panic 失败），优于上轮 Dogwood（S4=0）。**局限**：测试仅覆盖 4 文件所在包（全量测试依赖下载部分超时）；C-04 需人工审查 crush 架构后闭合。
