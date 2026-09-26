# 00 Overview — E2B Knowledge Archaeology

> run_id: `ARCH-2026-09-23-001` · skill v3.2 · commit `ccaf9fc`

## 一句话定位

**E2B = "在云端安全隔离沙箱中运行 AI 生成代码" 的开源基础设施的客户端形态。** 本仓库不包含服务器运行时（托管服务在 api.e2b.dev），发行本体是三套 SDK（TypeScript / Python sync+async / Code Interpreter 高层）+ CLI，契约由 `spec/`（Copybara 从 e2b-dev/runtime 与 e2b-dev/belt 同步）定义。

## 本次考古让我们认识到了什么（核心命题）

1. **沙箱即资源对象**：E2B 把"一个可执行 AI 代码的隔离环境"抽象成与数据库实例同级的托管资源——生命周期（create/connect/fork/pause/kill/snapshot）、配额（timeout 上限 Pro 24h / Hobby 1h）、可寻址（`${port}-<sandboxId>.e2b.app`）。
2. **秘密不落地的权威链**：从 API key 到 envd 令牌到流量令牌到 IAM 工作量令牌，全部以"占位符 → 平台侧按请求签发"的方式流动；SDK 层用 Proxy 陷阱与字符集校验把"把秘密泄漏进 header"的路径逐一封死。
3. **失败归因是 SDK 的核心工程质量**：连接中断后用健康探测区分"沙箱被杀"与"瞬时网络故障"，错误码分派（502→超时、429→限流、409→已暂停）让调用方能采取正确动作；两阶段超时（握手/执行）与并发信号量构成连接治理骨架。
4. **规范所有权即安全边界**：`x-internal` 标记在 envd.yaml 中声明，沙箱代理从标记自动生成拒绝列表——安全边界由"声明"而非"各处硬编码"维护；spec/ 由 Copybara pin 同步，仓库内禁止手改。

## 考古范围（按 skill v3.2）

| 层 | 产物 | 数量 |
|---|---|---|
| Project Layer | 01_project-layer.md | 1 地图 |
| Engineering Knowledge | 02_engineering-knowledge.md（EK Graph） | 32 EK |
| Generalized Knowledge | 03_knowledge-layer.md（KO 簇） | 9 KO |
| Flow Atlas | 04_flow-atlas.md | 7 类流 |
| Candidates | 05_candidates.md | 7 项 |
| Validation | 06_validation.md | 6 Auditor + Reconciliation |

## 关键数字（来自仓库事实）

- commit `ccaf9fc0ffe6ac39c7ec786af7608ab1de19467b`（[skip ci] Release new versions，2026-09-18）
- js-sdk / python-sdk 版本 2.51.0；pnpm@10.34.5 monorepo
- 测试 169 个 `*.test.ts`（vitest，live sandbox 60s 预算）
- 默认参数：requestTimeout 60s / retries 3 / sandbox 300s / keepalive ping 50s / 连接 100 / inflight 1000
- envd 端口 49983（REST）/ Jupyter 端口（code-interpreter，JUPYTER_PORT）
- 签名 URL 前缀 `v1_`；IAM 占位符 `${e2b.identity.tokens.<name>}`；Secret 占位符 `${e2b.secrets.<name>}`

## 质量指标速览（详见 06_validation.md）

| 指标 | 值 |
|---|---|
| EK 数 | 32 |
| EK 平均出边数 | 2.2（≥1 合格） |
| 游离 EK 比例 | 0%（32/32 全部连入 KO 簇） |
| 聚合规则覆盖率 | 100%（9/9 KO 有 aggregation_rule） |
| KO 平均簇规模 | 4.1（3~12 健康） |
| 盲重建 | performed（独立 Auditor 重读仓库，见 06） |

## 交付状态

- 本包仅含考古结果；目标仓库、KnowlegeMap、生产 Skill 均未修改。
- Corpus 提交：projects/e2b/（独立 PR）。
