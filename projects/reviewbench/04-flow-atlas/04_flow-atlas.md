# 04 · Flow Atlas（七类流）— ReviewBench（ARCH-2026-10-17-001）

> 全部从真实代码/文档导出；关键 Edge 标注 symbol/file/§节，可回溯。

## 1. Control Flow（评测控制流）
```
C1 onboarding：agent manifest（agents/*.json）→ schema 校验（schema/validate-manifest.mjs）→ test run（RUNNER.md, 25 PRs）
    → official 3-runs（§7.7）→ leaderboard
C2 eval 主流程：candidate findings → input-validation（input-validation.ts，strict 默认 true 硬失败 / --no-strict 警告 / --allow-empty 允许空跑）→ dedup/merge（golden/pipeline.ts）
    → resolveForScoring（lib/match-types.ts）→ unmatched 分类前 manifest/golden identity guard（repo/pr_number/head/base 全等检查，eval.ts）
    → classifyFindings（golden/classify-adapter.ts）→ scorePR/aggregate（scorer.ts）→ printSummary（print-summary.ts）
C3 本地运行：try-agent.sh（fresh container per PR）→ findings 目录 → npm run judge（JUDGING.md）
```
关键 Edge：C2 中 `evalPR`（eval.ts:364）→ askJudge（judge-call.ts，`JUDGE_CALL_TIMEOUT_MS`）→ checkpointFingerprint（checkpoint-fingerprint.ts）→ outputSidecarPath（output-paths.ts）。

## 2. State Flow（匹配/计分状态机）
```
raw findings → [matched: candidate→goldenIdx]（many-to-many）
   ├─ claimed for precision（candidate 按序认领第一个未认领 golden, §6.2.4）
   └─ unmatched → [classifier TP/FP]
→ scorer 内部状态：{G, G_TP, C, M, M_TP, U, U_TP, U_FP}（§7 定义）
→ 3 runs 独立计分（run 级保留, §7.7）
```
关键状态转换：`matched_candidates: Map<number,number>`（scorer.ts:125）——"matched→claimed"转换是防抢 credit 的唯一路径；`covered_golden` 集合用于 recall 去重计数。

## 3. Data Flow（数据管道）
```
D1 golden 构建：manifest（base/head）→ extraction（EXTRACTION.md：human comments + LLM passes + semgrep）
    → normalization（§4.3）→ classifier labeling（§5.4）→ human audit（47 修正）→ dedup（同 matcher）→ golden/*.json
D2 评测：agent findings → validation → matching（matcher.ts）→ unmatched classification → scoring
D3 manifest 数据：corpus/manifest.json（219 entries, list）→ try-agent.sh / judge 消费
```
关键事实：`golden-producers.ts` 从 `golden/` + `corpus/test/test.json` 可再生 §5.5 分布表（`npm run report:producers`）。

## 4. Evidence Flow（证据链）
```
E1 finding 证据：每条 finding 含 tp_fp/severity/category justification（golden/axios_axios_10836 样本）
E2 classifier 证据：prompt 公开（scripts/classifier/prompts.ts）+ agreement 报告（96.6%/98.7%/79.7%）
E3 matcher 证据：raw correspondences 保留审计（§6.2.3）+ matcher accuracy 随结果报告（§8）
E4 复现证据：checkpointFingerprint + 8 项 artifacts（§9）
```
关键性质：证据与标签同构（每个标签字段带 justification），使"标签可争议可修订"（§11 Feedback）。

## 5. Authority Flow（权威流）
```
A1 环境权威：镜像 digest pin（AGENT_CONTRACT）→ container 内 RB_* env 是唯一输入通道
A2 网络权威：egress 白名单（manifest egress[]）→ 只有声明域可达
A3 凭据权威：secrets env/files（manifest）→ 最小注入
A4 结果权威：offline 结果不可提交（OFFLINE_EVAL.md）→ 只有 portal 官方 run 可上 leaderboard
```
关键事实：schema required `image`（digest 格式）；`notes: "Reference manifest. Not a real agent."`（example-reviewer.json）表明 manifest 是可声明的治理面。

## 6. Memory Flow（状态隔离）
```
M1 PR 快照记忆：base/head 锁定（§3.2）→ head 后 commit 不可见（HARNESS.md §2）
M2 run 隔离记忆：fresh container per PR（AGENT_CONTRACT）→ 无跨 PR 状态
M3 文件记忆：每 run 独立目录（OFFLINE_EVAL.md: `run=$root/runs/my-reviewer-$(date...)`）→ findings/scores 不混
```
关键性质：评测的内存层是"无记忆"——以无状态换取结果独立性。

## 7. Policy Flow（治理闭环）
```
P1 计分政策：3 runs 取均值（§7.7）→ 决策：submission = mean，方差可审计
P2 聚合政策：macro + micro 双报（§7.6）→ 决策：两类问题（典型 PR vs 典型 finding）都可回答
P3 分层政策：severity/category 默认分层（§7.3）→ 决策：agent 策略差异可见（security-focused 用例）
P4 提交政策：offline 不可提交（EK-17）→ 决策：对外宣称必须官方 run
P5 校准政策：classifier 校准 → 重算下游指标（§5.3）→ 决策：校准不改则指标不更新
P6 分类器文化政策：prompt 编码团队 taste（§8）→ 决策：接受者须理解标签主观边界
```
治理闭环：Decision（P1-P6 定义）→ Approval（§9 artifacts 固定）→ Policy（METHODOLOGY.md 书面化）→ Enforcement（input-validation + schema + offline 隔离）→ Future Decision（§11 反馈修订 golden）。

## Flow→KO 交叉校验
- KO-01 ← EK-01/04/05/17：grounded/augmented 分离在 Control(C2)/State 流中可回溯（scorer.ts 公式 + §6.2.4 认领规则）。
- KO-03 ← Authority 流 A1-A4 + Memory 流 M1-M3：fresh container/digest pin/offline 隔离全部是运行机制而非文档声明。
- KO-04 ← Evidence 流 E2 + Policy 流 P5/P6：classifier 校准约束对应 judge-call/classify-adapter 实现路径。
- KO-05 ← Control 流 C2：matcher 判据（matcher.ts:43）在 eval 主流程中是匹配与 dedup 的共享路径。
- KO-06 ← Evidence 流 E4 + Policy 流 P1：checkpointFingerprint 与 3-runs 保留在 eval.ts/print-summary.ts 落地。
