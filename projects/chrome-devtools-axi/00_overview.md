# 00 · Overview — chrome-devtools-axi

> Run `ARCH-2026-11-26-001` · Job `KA-2026-11-26-001` · mode `initial` · priority `A` · 2026-11-26

## 项目一句话

`chrome-devtools-axi` 是 `chrome-devtools-mcp` 的 AXI 兼容 CLI wrapper：用短命 CLI + 持久 loopback bridge 把 MCP 的浏览器工具包装成 agent 友好的 TOON 输出，目标是 token 效率与 agent ergonomics。`VISION.md` 明确“full functional parity with chrome-devtools-mcp”，不新增上游没有的浏览器功能。

## 为什么今天选它

- **NEW**：Corpus 无该项目；此前只在 KnowlegeMap 的 P2 安全卡中出现。
- **真实安全边界**：有 GHSA `GHSA-x439-jhfh-v9x2`、CVE 描述、修复 commit `4ee5a49`、release `0.1.27`，以及现行 Host/Origin gate + 796 个执行测试。
- **单日可闭合**：17,051 行 TS、50 个 `.ts`、78 tracked files；体量足够小，能在一天内完成宽底座 + 窄尖顶 + 独立验证。
- **不是 star 排名**：同日候选 `openclaw`（refresh，gateway 级太大）、`chrome-devtools-mcp`（约 186k 行，太大）、`playwright-mcp`（3 天内已做）、`praisonai`（大型多 agent 框架）都被排队。

## 关键发现（先看这 5 条）

1. **Bridge 是 loopback 无认证控制面**：`POST /call` 把任意 MCP 工具名直接映射到 `client.callTool`；`roots` 由请求体提供，任意绝对路径，无服务端 allowlist。Host/Origin gate 只挡浏览器侧 DNS rebinding，不提供 caller 身份。这是本轮最重要的工程事实（KO-02）。
2. **身份 epoch 是 fail-closed 的**：chrome-devtools-mcp reconnect 后所有 page id 重发；AXI 不按 URL/标题/`[selected]` 自动 retarget，而是清空 selection、返回 identity error、要求重新选择（KO-04）。
3. **新鲜度不是时间**：每次 snapshot 递增持久 `generation`，页面侧 `MutationObserver` 计突变；`generation` 不等或 `mutations>0` 就 `STALE_REF`。legacy 无 `gN:` 前缀的 ref 是明确例外（KO-05）。
4. **隔离跟随 custody**：keychain flag 只在 AXI 自己 launch 浏览器时追加；attach/autoConnect/browserUrl/wsEndpoint 不追加。session 只隔离 bridge，多 session attach 同一浏览器仍会 cross-talk（KO-03）。
5. **质量/发布有机器 custody**：CI Node 24 + frozen lockfile + build/test；airlock 用 pnpm lint；guard-generated-files 禁止手改 CHANGELOG/manifest；release-please + npm `--provenance`。但这是本仓库治理，不跨项目推广（KO-08，OVER_GENERALIZED）。

## Package 地图

| 文件 | 内容 |
|---|---|
| `00_overview.md` | 本页：项目、选择理由、关键发现、指标、验证结论 |
| `01_project-layer/project-map.md` | 项目定位、架构图、36 个命令、模块责任、状态、生命周期、配置、权限治理、边界 |
| `02_engineering-knowledge/engineering-knowledge.md` | 36 条 EK，每条 claim/category/abstraction/value/status/detail/scope/evidence/links |
| `02_engineering-knowledge/ek-graph.yaml` | 36 nodes + 59 声明边；无孤立 |
| `02_engineering-knowledge/ek-metrics.json` | EK 指标（机器可读） |
| `03-knowledge-layer/knowledge-objects.md` | 8 个 KO：claim、derivation、aggregation_rule、scope、counterexample、validation、flow traceability |
| `03-knowledge-layer/ko-graph.yaml` | 8 KO YAML 图谱 |
| `03-knowledge-layer/ko-metrics.json` | KO 规则/簇/层级统计 |
| `04_flow-atlas/flow-atlas.md` | 7 类流、48 条边、关键路径读数、未覆盖边界 |
| `04_flow-atlas/flow-atlas.yaml` | 机器可读 48 edges |
| `05_candidates/candidates.md` | 12 个候选：hypothesis / uncertain_pattern / known_limitation / evidence_gap / cross_project_candidate |
| `06-validation/evidence-index.md` | 49 条 Evidence，回到文件/行号/commit |
| `06-validation/independent-validation.md` | Independent Findings、KO 对比、3 成功 + 3 错误、7 遗漏、Benchmark 5/6 |
| `06-validation/reconciliation.md` | R-01..R-07 修正记录 + 校验清单 |
| `06-validation/quality-metrics.json` | 机器可读质量指标 |
| `run_metadata.yaml` | run/repository/commit/skill version/timestamp/metrics/validation |

## 质量指标

| 指标 | 值 |
|---|---|
| EK | 36 |
| EK 孤立 | 0（36/36 有 links） |
| EK 声明边 | 59（平均出边 1.33） |
| KO | 8（R1×1 / R2×3 / R4×4） |
| KO `aggregation_rule` | 8/8 |
| KO `facts == cluster` | 8/8 |
| Flow edges | 48（7 类） |
| Evidence rows | 49 |
| Candidates | 12 |
| 测试 | 29 files / 654 static `it(` / 796 executed，全部通过，80.57s |
| Build | `pnpm run build` 通过；`git status --porcelain` 空 |
| Validation | CONFIRMED 4 / PARTIALLY 2 / DOWNGRADED 1 / OVER_GENERALIZED 1 / CONTRADICTED 0 / MISSING 7 |
| Gate | **PARTIAL**（静态 + 本机测试通过；无 live Chrome/运行时攻击/跨 UID） |
| Skill Evolution | **无**（本轮未暴露系统性 skill 缺陷；只有项目级证据缺口） |

## 边界（必须一起读）

- 本轮 **没有启动真实 Chrome**、没有 live CDP、没有运行中的 bridge 攻击请求；所有 runtime claim 只到“代码 + 本机测试”层。
- 目录/文件权限、跨 UID loopback 可达性、HTTP/2 `:authority`、重复 `Host`、代理改写、`isBridgeProcess` 伪造/PID 复用、真实 MCP roots 文件范围、npx 供应链均未测，全部留在 Candidates / MISSING。
- 目标仓库只读；KnowlegeMap / 生产 Skill 未修改；未写飞书（无用户显式授权）。

## 下一步（下一次 refresh 优先级）

1. 补 M-01：跨 UID loopback 可达性实验（一次性 VM）。
2. 补 M-02：重复 Host / HTTP2 `:authority` / 反代改写。
3. 补 M-04：live MCP roots 文件读写范围。
4. 补 M-05：`isBridgeProcess` 伪造 / PID 复用。
5. 观察 `openclaw` refresh（CVE-2026-28458 Browser Relay `/cdp` 无 token）与 `chrome-devtools-mcp` upstream 对照。
