# 06 Validation / Coverage Gaps（ARCH-2026-11-16-001）

## 1. 未覆盖的仓库区域
| 区域 | 状态 | 原因 | 影响 |
|---|---|---|---|
| 真实 LlamaIndex 集成 | 未运行时验证 | `llama-index-core` 未安装；`test_integration_llamaindex.py` skip | 读旁路在 fake `BaseChatStore` 复现；真实 `ChatMemoryBuffer` 可见性未证 |
| 真实 Agno 集成 | 未运行时验证 | `agno` 未安装；测试 skip | 只按源码记录 `_store.get` 回填路径 |
| HTTP server 端到端 | 未启动 | 本轮只做静态源码 + 单元级探针 | `/reset` 无认证、`/read` found 语义未在真实端口验证 |
| Python 3.9–3.11 / 3.13–3.14 | 未执行 | 本机只有 3.12 | identity contextvars/opcode 的跨版本兼容未证 |
| Benchmark / 性能 | 未复跑 | 0.4.0 无现成复现命令；README 数字 pin 0.2.2 | 不能声称 0.4.0 达到 92.5% recall / 62 µs |
| PyPI provenance | 未核验 | 无构建记录 | tag 含修复 ≠ PyPI 产物含修复 |
| `mcp-server/` | 未深入 | 只包扫描器，不触 guard 状态 | 可能遗漏其工具参数/权限面 |
| `detectors/` 全部 11 个 | 只覆盖默认 7 + 4 非默认名称 | 非默认 detector 未逐条行为测试 | ToolAbuse/PrivilegeEscalation 等默认关闭，实际防护面未证 |
| 旧 KO 逐条对照 | 未做 | 旧 corpus 只读标题/概述，未逐条正文比对 | BC-01 FAIL；refresh 连续性缺口 |

## 2. 未覆盖的攻击族（项目自报 xfail）
`tests/benchmarks/test_delayed_activation.py` 等 10 个 xfail 覆盖：跨写凭证拼接、time-bomb、跨 key SQL 片段、multi-write SQL 累积、base64 指令、chained tool outputs、Unicode homoglyph、read-then-write taint、非英文注入、fake error message injection。
**本轮不把它们当作已防护**；只作为 F 层待测清单。

## 3. 未覆盖的边界
- `set_messages` 索引错位 / 全拦旧消息残留（C-16/C-18）：源码可推导，未跑完整序列。
- `retire_if` predicate 异常（C-17）：未注入异常做端到端。
- `/read` `found=None`（C-19）：未启动 server 写 `None`。
- `events` 无上限的长期内存增长：未做 soak test。
- `quarantine` dict 无上限：未做大量注入的容量测试。
- `SnapshotStore` 淘汰策略：只确认 `_max=50`，未验证满 50 后淘汰顺序/审计影响。

## 4. 待 Owner / 下一轮
1. 11-20 三项实测：注入复现、delete 回执一致性、作用域串读；本轮已给出可执行入口（fake Llama 脚本、`delete_messages`/backing 对照、`set_messages` 索引序列）。
2. 真实 LlamaIndex/Agno 安装后复跑集成测试，确认读旁路与 `.guard` 可见性。
3. 核验 PyPI 0.4.0 构建 commit；决定 0.4.1 或 CHANGELOG 修正。
4. 下一轮 refresh 加 `refresh-delta.md`（HR-06），先补旧 KO 逐条状态。
