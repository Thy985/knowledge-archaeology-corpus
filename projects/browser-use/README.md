# browser-use — Knowledge Archaeology

**一句话定位**：开源浏览器 Agent 库，1.0.8 起把动作空间换成“持久 Python REPL + raw CDP”，默认 runtime 是 Browser Use Cloud。

## 当前条目（latest）

- 考古日期：2026-11-27（`ARCH-2026-11-27-001`，mode: `refresh`）
- 目标版本：`1.0.8 @ 753d629d76eb2b670f4cf467f3355bbe750f6845`（2026-11-21）
- 基线：`0.13.10 @ d8110c5`（`ARCH-2026-09-22-001`，initial）
- Skill：knowledge-archaeology v3.2
- 产物：`archaeology-runs/ARCH-2026-11-27-001/`（完整 00–06 + snapshot + job manifest + run metadata）

**量级**：55 EK / 12 KO（10 L3 + 2 L4，0 L5）/ 7 类 Flow / 59 Flow Edge / 13 Candidates。本地 964 个非 Chrome 测试通过，3 skipped，140 chrome-marked deselected。

**独立验证**：9 CONFIRMED / 3 PARTIALLY_CONFIRMED / 0 CONTRADICTED；3 个 material error 已修正（step 默认值、judge→is_successful 错误边、attach approval 过度表述）。

### 1.0.8 关键认知

1. **代码即动作空间**（KO-01）：DOM serializer、typed action schema、action registry 被删除；built-ins 只剩 `python` / `finish` / `finish_from_repl`。
2. **三平面执行**（KO-02）：parent 持 key/budget/delivery；worker 以用户身份跑代码、可被杀/重启；browser-harness daemon 持浏览器、可跨进程存活。
3. **预算→交付阶梯**（KO-03）：90% 预算进入 delivery-only；cell timeout 杀进程树，文件/浏览器状态保留，Python 变量丢失。
4. **交付是结构契约**（KO-04）：split / placeholder / missing-file 各修一次；只校验结构，不校验语义。
5. **两条 verdict**（KO-05）：`is_successful()` 只读 agent 的 `finish(success=...)`；judge 是同一模型、同一 trace 的第二次请求，cached path 还共享 prefix，不是独立验证。
6. **redaction 不是 sandbox**（KO-06）：worker 可读用户文件、parent `.env`、`/proc/<pid>/environ`；截图和 `js()` 不 redact。
7. **domain allowlist 不是网络隔离**（KO-07）：只拦 main-frame document；iframe/subresource/Python HTTP 不过滤。
8. **local debug port 是 ambient authority**（KO-08）：`127.0.0.1`、无 token、同机账户可 attach；profile copy 带 cookies。
9. **双 compaction 路径**（KO-09）：server-side `LazyCompaction` vs client-side summary checkpoint + 20/2 image prune。
10. **managed cloud 是默认运行时**（KO-10）：`Browser()` cloud + `bu-4.0` + 单 key；local 是显式 opt-in。
11. **L4 模型**（KO-11/KO-12）：action space 决定治理面；confidentiality / execution containment / navigation policy / authority grant 是四种不同边界。

## 历史条目

`archaeology-runs/ARCH-2026-09-22-001/` 保留 0.13.10 原始产物（manifest / snapshot / metadata / independent validation）。旧顶层 `00_overview.md`–`06_validation/` 仍是 0.13.10 版本，未覆盖；新 run 的完整包在 `archaeology-runs/ARCH-2026-11-27-001/`。旧 metadata 也已复制到该历史 run 目录，避免元数据丢失。

### 0.13.10 基线认知（历史，不适用于 1.0）

1. 软门控 vs 硬门控：循环检测只注入 nudge，预算/失败用不可绕过终止。
2. 上下文预算分级降级：compaction → 75% 警告 → 最后一步强制 done。
3. 事件驱动 watchdog：bubus EventBus + 16 监视器。
4. 三重安全边界：域白名单 + 重定向捕获 + `block_ip_addresses`。
5. OSS 漏斗产品化：与 mem0 同构（OSS 库→云端→自有模型推荐）。

## 文件

| 路径 | 内容 |
|---|---|
| `metadata.yaml` | 项目级元数据（latest + runs + previous baseline） |
| `archaeology-runs/ARCH-2026-11-27-001/00_overview.md` | 1.0.8 概览 |
| `archaeology-runs/ARCH-2026-11-27-001/01_project-layer/` | 项目地图 / 0.x→1.0 delta |
| `archaeology-runs/ARCH-2026-11-27-001/02-engineering-knowledge/` | 55 EK |
| `archaeology-runs/ARCH-2026-11-27-001/03-knowledge-layer/` | 12 KO |
| `archaeology-runs/ARCH-2026-11-27-001/04-flow-atlas/` | 7 flows / 59 edges |
| `archaeology-runs/ARCH-2026-11-27-001/05-candidates/` | 13 candidates |
| `archaeology-runs/ARCH-2026-11-27-001/06-validation/` | independent findings + validation + reconciliation |
| `archaeology-runs/ARCH-2026-11-27-001/snapshot_artifact.md` | snapshot |
| `archaeology-runs/ARCH-2026-11-27-001/job_manifest.md` | job selection |
| `archaeology-runs/ARCH-2026-09-22-001/` | 0.13.10 baseline |

## 连接

- `playwright-mcp`：typed MCP tools vs code-execution REPL，两条 Computer Use 控制面。
- `chrome-devtools-axi`：DevTools wrapper/upstream 与本地端口暴露。
- `openclaw`：Browser Relay `/cdp` 无 token，同族本地控制面。
- `graphiti` / `mem0` / `agent-memory-guard`：记忆压缩、OSS 漏斗、记忆写入治理的对照点。
