# MCP Go SDK · Knowledge Archaeology

> **对象**：`modelcontextprotocol/go-sdk`（MCP 协议的**官方 Go 实现**）
> **区分**：本目录是**实现侧**；`projects/mcp/` 是**规范侧**（`modelcontextprotocol/modelcontextprotocol`，TypeScript）。两者不是同一仓库。
> **考古日期**：2026-11-07（Asia/Shanghai）
> **Run**：`ARCH-2026-11-07-001`（mode: initial）
> **Commit**：`df0f29525bc6c590b6b68d9aae581ec0f21ebb3e`（main, 2026-11-06 23:38 +0800）
> **Skill**：knowledge-archaeology v3.2

## 一句话定位

MCP 官方 Go SDK 在 2026-07-28 协议从"有状态双向会话"转为"无状态 request/response"的转折点上，用**双协议栈 + 探测优先 + 无损回退**完成迁移，同时保留 12 个月弃用窗口；其最有价值的工程知识不是"实现了哪些方法"，而是**如何让两代协议在同一个 handler 里共存**。

## 产物清单

| 文件 | 内容 |
|---|---|
| `00_overview.md` | 定位、七个问答、关键发现、三层配比 |
| `01-project-layer/` | 架构 / 生命周期 / 数据模型 / 配置与测试 / 治理 |
| `02-engineering-knowledge/` | Evidence Index（53 Facts）、EK-01~30、EK-31~33（Reconciliation 追加）、EK Graph |
| `03-knowledge-layer/` | KO-01~08、KO 聚合矩阵 |
| `04-flow-atlas/` | 七类流 29 Edge |
| `05-candidates/` | 7 条未验证假说 |
| `06-validation/` | 内部验证、反例、Reconciliation |
| `archaeology-runs/ARCH-2026-11-07-001/` | Job Manifest、Snapshot、原始 Package、独立验证报告、run metadata |

## 质量速览

| 指标 | 值 |
|---|---|
| Facts | 53 |
| EK | 33（30 + 3 条 Reconciliation 补入） |
| KO | 8 |
| Candidates | 7 |
| Flow Edge | 29 |
| EK 平均度 | 2.61 |
| 孤立 EK | 0 |
| 聚合规则覆盖 | 8/8 |
| 独立审计 | 14 CONFIRMED / 6 PARTIAL / 1 DOWNGRADED / 1 OVER_GENERALIZED / 3 MISSING / 0 CONTRADICTED / 2 NEEDS_HUMAN_REVIEW |

## 核心 KO（速览）

1. **KO-01** 协议迁移：探测优先、失败回退（L3）
2. **KO-02** 无状态化 = 请求级会话，去掉的是传输层 session ID（L4）
3. **KO-03** 会话身份绑定是传输层不变量（L3）
4. **KO-04** 版本是三平面事实（header/`_meta`/握手）（L3）
5. **KO-05** 门禁有效性取决于执行点（L4，pending）
6. **KO-06** 重放依赖"不得部分重放"契约（L3）
7. **KO-07** 服务器主动请求折叠为多轮结果（L3）
8. **KO-08** 负向清单降低误用，但不保证治理一致性（L3）

## 读法建议

- 先读 `00_overview.md` 的"七个关键问答"，再按 KO 索引回到底层 EK。
- 关注"声明 vs 实现"差距时，直接看 `06-validation/reconciliation.md` 的 §1 与 `05-candidates/`。
- 想对比"规范 vs 实现"，配合 `projects/mcp/` 的规范侧考古阅读。

## 未覆盖（诚实说明）

`auth/authorization_code.go`、`auth/extauth/*`、`oauthex/*`、`internal/jsonrpc2/conn.go` 仅做结构级阅读，未逐行提取 EK；覆盖率口径见 `05-candidates/candidates.md` C-05。
