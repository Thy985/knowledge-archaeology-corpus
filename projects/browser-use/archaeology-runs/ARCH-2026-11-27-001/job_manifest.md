# Job Manifest — ARCH-2026-11-27-001

- **job_id**: `KA-2026-11-27-001`
- **run_id**: `ARCH-2026-11-27-001`
- **date**: 2026-11-27（Asia/Shanghai）
- **project**: `browser-use`
- **repository**: https://github.com/browser-use/browser-use.git
- **mode**: `refresh`
- **priority**: `A`
- **baseline run**: `ARCH-2026-09-22-001`（0.13.10 @ `d8110c5`）
- **target ref**: tag `1.0.8` @ `753d629d76eb2b670f4cf467f3355bbe750f6845`（2026-11-21 19:30:22 -0800）
- **source_entry**:
  - `07_radar_watch/current_focus.md`（Computer Use / Browser，2026-11-27 Day5 验证日）
  - `05_scan_logs/2026-11/2026-11-27_radar-watch_scan.md`（browser-use 1.0.8 源码级验证）
  - `03_expansion_queue/candidates/[cand]computer-browser-use-2026.md` §3.9 + §5（validated 源码级）
  - `03_expansion_queue/candidates/[cand]local-control-plane-exposure-2026-11.md`（P1/P7 本地控制面问题框架）
  - `08_agent_centric/domains/agent-computer-use.md`（A~H 能力树与判断 9/10）
  - Corpus `projects/browser-use/metadata.yaml`（旧版 0.13.10 基线）

## 候选排序（不按 star）

| 候选 | 状态 | 本次判断 | 决策 |
|---|---|---|---|
| `browser-use` | `REFRESH_REQUIRED` | 2026-11-22 起 1.0.8；1.0 是“REPL + raw CDP、无 DOM serializer”的架构级重写；旧基线 0.13.10 与现版对象不同；今日雷达已完成源码级四问并升 validated；仓库仅 35,428 行 Python、25 个 CI 测试文件，可在单日闭合；与既有 corpus 可做版本对照 | **选中** |
| `@playwright/mcp` | `RECENTLY_COMPLETED` | 0.0.86 今日已验证，但 2026-11-23 刚完成 initial（PR #80 尚未合并）；4 天内重复深度考古边际收益低 | 排队 |
| `openclaw` | `REFRESH_REQUIRED` | 旧基线 2026.9.4，npm 已到 2026.11.10，CVE-2026-28458 涉及 Browser Relay `/cdp` 无 token；但 gateway monorepo 范围远大于单日，且今日雷达主体不是它 | 排队（下一候选） |
| `chrome-devtools-mcp` | `NEW` | 与 Axi wrapper/upstream 对照价值高；队列记录显示约 18.6 万行 / 420 文件，单日无法同等深度闭合 | 排队 |
| `webwright` | `OBSERVATION` | 0.0.7 包级仅证明“脚本 + diff 即产物、无执行记录”，不足以支撑完整项目考古 | 维持观察 |
| `praisonai` | `NEW` | `browser start` 默认 `0.0.0.0` 是 P3 证据，但属大型多 agent 框架，范围不适合今日唯一名额 | 排队 |
| `local-control-plane-exposure-2026-11` | `EVIDENCE_ONLY` | 跨对象归纳卡，不是可克隆仓库；作为问题定义与 Benchmark 评分表 | 不选 |

## 选择理由

1. **对象已经变了**：1.0 不是 0.x 的增量，而是删除 DOM serializer / action schema、改为持久 Python REPL + raw CDP。旧 KO（软门控、16 watchdog、DOM stale 防护、域白名单）不能直接继承，必须 refresh。
2. **基准学习价值最高**：同一项目、同一 corpus 目录、跨两个架构范式，可检验“哪些工程不变量跨架构保留，哪些是 0.x 偶然实现”，是 Abstraction Auditor 的天然对照。
3. **今日雷达已给一手入口**：`05_scan_logs/2026-11/2026-11-27` 给出包级/源码级四问与安全侧证据；本轮可把“雷达验证”升级为“完整仓库考古”，并独立复核，而不是重复它的结论。
4. **体量可闭合**：35,428 行 Python（含 18,492 行 tests）、146 tracked files，适合在单日完成 Discovery / EK / Flow / KO / Independent Validation 全链。
5. **边界清晰**：默认云浏览器 + 本地 Chrome 两条路径并存；permission / secret / workspace / telemetry / judge 都有明确代码与测试，不靠文档推断。

## 本轮必须回答的问题

1. 1.0 的“无中间层”究竟删掉了什么、保留了什么？控制流是否从 action schema 变成“模型写 Python + REPL 执行 + finish 交付”？
2. 持久 REPL 的进程边界、超时、环境 allowlist、secret 注入/redaction、workspace 隔离，哪些是硬边界，哪些只是显示层脱敏？
3. 本地 Chrome 的默认端口/认证/attach 行为，是否仍构成“同机任意账户可接管”的本地控制面？
4. judge（成功判定）与 `finish(success=...)` 的权威关系：判定依据来自执行事实，还是同一模型自评？
5. 从 0.13.10 到 1.0.8，哪些旧 KO 应保留、哪些应降级为“0.x 实现”、哪些应新立？跨架构不变量必须有代码证据。

## 边界与只读声明

- 目标仓库只读；本轮只写 `archaeology-jobs/ARCH-2026-11-27-001/` 与 corpus 分支。
- 不覆盖 `projects/browser-use/archaeology-runs/ARCH-2026-09-22-001/` 的任何历史文件；本轮新增 `ARCH-2026-11-27-001/` 并更新项目顶层索引。
- 飞书落盘未获显式授权，默认不做。
- 不在目标仓库跑会写文件的命令；如跑测试，先记录工作树状态，测试后核验 `git status --porcelain` 为空。
