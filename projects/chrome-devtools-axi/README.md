# chrome-devtools-axi · Knowledge Archaeology

一句话：`chrome-devtools-axi` 是 `chrome-devtools-mcp` 的 AXI 兼容 CLI wrapper，用短命 CLI + 持久 loopback bridge 提供 agent 友好的 TOON 浏览器工具输出。

## 本次考古

- Run: `ARCH-2026-11-26-001`
- Mode: `initial`
- Commit: `1dbdac1ca1122e1d2b76d889bbc8a2dd2eb2dcd7` (`chrome-devtools-axi-v0.1.46-2-g1dbdac1`)
- Skill: `knowledge-archaeology v3.2`
- Date: `2026-11-26`
- Validation: `PARTIALLY_CONFIRMED` / Gate `PARTIAL`
- Runtime verified: `false`（无 live Chrome / live CDP / 运行中 bridge 攻击）

## 产物索引

| 目录/文件 | 内容 |
|---|---|
| `00_overview.md` | 项目定位、选择理由、关键发现、指标、边界 |
| `01_project-layer/project-map.md` | 项目地图：架构、入口、36 命令、模块、状态、生命周期、配置、治理、依赖 |
| `02_engineering-knowledge/engineering-knowledge.md` | 36 条 EK（claim/detail/scope/evidence/links） |
| `02_engineering-knowledge/ek-graph.yaml` | 36 nodes / 59 声明边 / 0 孤立 |
| `03-knowledge-layer/knowledge-objects.md` | 8 个 KO（aggregation_rule / derivation / flow traceability / validation） |
| `03-knowledge-layer/ko-graph.yaml` | KO 机器可读图 |
| `04_flow-atlas/flow-atlas.md` | 7 类流 / 48 条边 |
| `05_candidates/candidates.md` | 12 个未确认候选 |
| `06-validation/evidence-index.md` | 49 条证据（文件/行号/commit） |
| `06-validation/independent-validation.md` | 独立重读、KO 对比、3 成功 + 3 错误 + 7 遗漏 + Benchmark |
| `06-validation/reconciliation.md` | R-01..R-07 修正记录 |
| `run_metadata.yaml` | run/commit/skill/metrics/validation |
| `archaeology-runs/ARCH-2026-11-26-001/` | 本次 run 的完整包快照（含 job manifest / snapshot artifact） |

## 核心结论

1. Host/Origin gate 只挡浏览器侧 DNS rebinding；`/call` 无 token/auth，`roots` 任意绝对路径，无 per-caller 审计 —— 本地控制面存在 confused deputy 面（KO-02）。
2. Reconnect 是 identity epoch：AXI 清空 selection、显式 pageId 返回 identity error、不自动 retarget（KO-04）。
3. Freshness = `generation` + `mutations`，legacy 无 `gN:` ref 是明确例外（KO-05）。
4. 隔离跟随 custody：keychain flag 只在 launch 追加；session 只隔离 bridge，attach 同一浏览器会 cross-talk（KO-03）。
5. 质量/发布由 CI/guard/release 机器强制，但只适用于本仓库治理（KO-08，OVER-ABSTRACTED / scoped）。

## 下一次 refresh 优先补

1. 跨 UID loopback 可达性。
2. 重复 Host / HTTP2 `:authority` / 反代改写。
3. live MCP roots 文件读写范围。
4. `isBridgeProcess` 伪造 / PID 复用。
5. live keychain / npx 供应链。

> 本目录不保存目标项目源码；所有事实均可回到 commit `1dbdac1` 的仓库文件/行号，或本机 `pnpm test` 输出（EV-046）。
