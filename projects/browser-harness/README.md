# browser-harness — Knowledge Archaeology

**Latest**: browser-harness 0.1.15 @ `373aecc4`（2026-11-25），run `ARCH-2026-11-28-001`（refresh）。
**一句话定位**：browser-use 出品的极简自愈浏览器 harness——LLM 经一根可编辑 CDP WebSocket 直连真实 Chrome，agent 在执行中自写缺失 helper；无框架、无 recipes、无 rails。README 现已把 Browser Harness 表述为 Browser Use 内部的 engine，`browser-use` CLI 为主入口。

## 本次（0.1.15）核心认知
- **Tab ownership 是 routing contract，不是 security boundary**：`tab_owner` / `session_owner` / `GUARDED` 三方法 / iframe 证明 / URL-title 清空 / idle 回收 / owner 状态持久化都已落地并有单测；但 daemon 接受任意 `owner` 字符串，不做认证。
- **两个旁路**：无 `BH_OWNER` 的 Claude 子 agent 只被拒 `close_tab`/`release_tabs`，仍可 `cdp()` 驱动父 current tab；`Target.attachToBrowserTarget` 的 sid 只进 `browser_sessions`，不进 `session_owner`，未知 sid 命中 `.get(sid, key)` 默认值而放行，browser-domain 方法可越出 tab 归属。
- **版本缝隙**：browser-use 1.0.8 钉 `browser-harness==0.1.13`，自身零使用 owner API；非 shared Browser 各自独立 daemon+浏览器，`Browser('chrome')` 走共享 default daemon 且无 owner 身份。上游能力未在下游生效。
- **0.1.15 未记录变更**：相对 0.1.14，CLI telemetry 不再出域脚本文本、stdout/stderr 尾、helper 参数与异常文本；CHANGELOG 无 0.1.15 段。

## 产物清单
| 目录/文件 | 内容 |
|---|---|
| `metadata.yaml` | 最新 run 元数据 + 历史 runs 索引 |
| `00_overview.md` | 0.1.15 概览与 headline |
| `01-project-layer/` | 三平面架构 / 模块 / 数据结构 / 生命周期 / 配置 / 治理 |
| `02-engineering-knowledge/` | 60 EK + 六类边 |
| `03-knowledge-layer/` | 10 KO（9 L3 + 1 L4，0 L5），R1–R4 聚合规则 |
| `04-flow-atlas/` | 七类流 |
| `05-candidates/` | 9 个候选（含旧 C-02 降级、browser-level CDP、版本缝隙、telemetry delta） |
| `06-validation/` | self-check / 独立盲重建 / 独立验证 / reconciliation / evidence |
| `archaeology-runs/ARCH-2026-11-28-001/` | 本次 run 原始产物（manifest/snapshot/package/metadata） |
| `archaeology-runs/ARCH-2026-10-04-001/` | 0.1.13 initial run 原始产物 + 旧 top-level package snapshot |

## 历史
| run | version | mode | date | validation |
|---|---|---|---|---|
| `ARCH-2026-10-04-001` | 0.1.13 @ `afbcc381` | initial | 2026-10-04 | 8 CONFIRMED / 2 PARTIAL / 2 DOWNGRADED / 0 OVER / 5 MISSING / 0 CONTRADICTED / 1 NEEDS_HUMAN_REVIEW |
| `ARCH-2026-11-28-001` | 0.1.15 @ `373aecc4` | refresh | 2026-11-28 | 79 CONFIRMED / 7 PARTIAL / 1 DOWNGRADED / 1 OVER_GENERALIZED / 6 MISSING / 3 CONTRADICTED / 3 NEEDS_HUMAN_REVIEW |

## 验证结论
`PARTIAL_PASS_WITH_DOWNGRADE`。核心结构判断保留；KO-09 由 L4 降为 L3；EK-10 的告警频次表述被纠正；EK-56 所引 README 句在本快照不存在，已改为 `SKILL.md:52` 的“raw CDP 可用”并保留代码缺口。无运行时探针，C-01/C-03/C-07 仍为 NEEDS_HUMAN_REVIEW。Skill Evolution 未触发。

## 关联
- 同厂对照：`browser-use` 1.0.8（2026-11-27 run）——上层 LLM 决策；本仓库为 CDP 连接层。
- 旧 connecting threads：E2E-CLI / campus_order(OpenClaw) / agent-attention / Computer Use；EP-002 权限边界。
