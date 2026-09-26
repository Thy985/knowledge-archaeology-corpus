# Job Manifest — ARCH-2026-09-22-001

## 候选排序（多因子，非 star 排名）
| 排名 | 项目 | ★ | 语言 | 最近 push | 知识价值 | Agent-AI 相关性 | Novelty | 增长/发布信号 | 与 Corpus 连接 | 已考古? | 品类轮换 |
|---|---|---|---|---|---|---|---|---|---|---|---|
| 1 | **browser-use/browser-use** | 115,743 | Python | 09-18 | A | 极高（campus_order 直接依赖） | 未考古 | 09-20 Anthropic Computer Use GA + Browser Use 官方工具化 | openclaw（已考古） | 否 | 09-21 记忆→09-22 computer-use ✅ |
| 2 | e2b-dev/E2B | 13,903 | Python | 09-21（今日） | A | 高（TeamMind/EP-002） | 未考古 | 今日 push | agent-governance-toolkit（部分覆盖） | 否 | 沙箱品类 |
| 3 | getzep/graphiti | 31,057 | Python | 09-21（今日） | A- | 高（记忆图谱） | 未考古 | 今日 push | mem0（昨日刚考古） | 否 | 连续两天同类，边际递减 |
| 4 | microsoft/Webwright | 6,014 | Python | 08-03 | B | 中 | 未考古 | 1.5 月无活动，增长弱 | computer-browser-use 卡 | 否 | — |

## 选中项目
- **job_id**: ARCH-2026-09-22-001
- **project**: browser-use
- **repository**: https://github.com/browser-use/browser-use.git
- **mode**: initial
- **priority**: A
- **reason**:
  - computer-use 主线开源事实标准（115,743★，Python/Playwright，MCP-native，Claude Code/Codex/Cursor/Hermes/OpenClaw 直接可用）；未考古
  - Agent-AI Engineering 相关性极高：用户生产项目 campus_order（OpenClaw）直接依赖 browser-use 生态；E2E-CLI 升级路线参照
  - 近期发布信号强：09-18 push；09-20 雷达重大事件（Anthropic Computer Use/Skills/Files API GA + 新 Browser Use 工具官方化 + Vercel Agent Browser + Browserbase Stagehand 重建 + Skyvern 3.0 验证码突破）
  - 品类轮换合理：09-21 记忆（mem0）→ 09-22 computer-use（browser-use）
  - 与已有 Corpus 连接：openclaw（已考古，浏览器 App 同架构）；computer-browser-use-2026 卡（A）
  - Benchmark 学习价值：web agent 的 DOM 提取→LLM 规划→action 执行→失败重试→telemetry 循环是 agent 工程核心范式
- **source_entry**: `03_expansion_queue/candidates/[cand]computer-browser-use-2026.md`（A 级）+ `[cand]browser-harness-2026.md`（A 级）
