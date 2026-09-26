# 00 Overview — E2B（corpus 版，Reconciliation 后）

> run_id: `ARCH-2026-09-23-001` · skill v3.2 · commit `ccaf9fc`
> 本文件为 Reconciliation 修正版；原始考古产物保留于 `archaeology-runs/ARCH-2026-09-23-001/package/`。

## 一句话定位

**E2B = "在云端安全隔离沙箱中运行 AI 生成代码" 的开源基础设施的客户端形态。** 本仓库不包含服务器运行时（托管服务在 api.e2b.dev），发行本体是三套 SDK（TypeScript / Python sync+async / Code Interpreter 高层）+ CLI，契约由 `spec/`（Copybara 从 e2b-dev/runtime 与 e2b-dev/belt 同步）定义。

## 本次考古让我们认识到了什么（核心命题）

1. **沙箱即资源对象**：E2B 把"一个可执行 AI 代码的隔离环境"抽象成与数据库实例同级的托管资源——生命周期（create/connect/fork/pause/kill/snapshot）、配额（timeout 上限 Pro 24h / Hobby 1h）、可寻址（`${port}-<sandboxId>.e2b.app`）。
2. **秘密不落地的权威链**：从 API key 到 envd 令牌到流量令牌到 IAM 工作量令牌，全部以"占位符 → 平台侧按请求签发"的方式流动；SDK 层用 Proxy 防护与字符集校验把"把秘密泄漏进 header"的路径逐一封死。
3. **失败归因是 SDK 的核心工程质量**：连接中断后用健康探测区分"沙箱被杀"与"瞬时网络故障"，错误码分派（502→超时、429→限流、409→已暂停）让调用方能采取正确动作；两阶段超时（握手/执行）与并发信号量构成连接治理骨架。
4. **规范所有权即安全边界**：`x-internal` 标记在 envd.yaml 中声明，沙箱代理从标记自动生成拒绝列表——安全边界由"声明"而非"各处硬编码"维护；spec/ 由 Copybara pin 同步，仓库内禁止手改。

## Reconciliation 说明（阶段 5 → 6）

| 项 | 修正 |
|---|---|
| EK-14 | **CONTRADICTED 修正**：RUNTIME_PROBED_PROPS 非"读取即抛"，实为"读取返回 stand-in、使用即抛"（iam.ts:118-146）；补 `has` trap（`name in iam.tokens` 只认自有键） |
| EK-19 | 证据路径修正：`packages/js-sdk/src/sandbox/commands/index.ts`（489 行），subsystem 归属为 sandbox 内 |
| EK-02 | 补充：onTimeout 未配置时 autoPause 字段整体省略（API 拥有默认值，与 C-03 呼应） |
| EK-13 | 补充：`SandboxIamTokenType = 'JWT-SVID'`（服务端类型集，may grow） |
| KO-02 | L4 降级确认：判别联合为 TS 静态类型特性，跨语言需运行时校验——保留 L4 但标注语言相关性 |
| Flow Atlas | commands 引用路径补全 |

## 考古范围与产出

| 层 | 产物 | 数量 |
|---|---|---|
| Project Layer | 01_project-layer/ | 1 地图 |
| Engineering Knowledge | 02_engineering-knowledge/ek-graph.md | 32 EK |
| Generalized Knowledge | 03_knowledge-layer/ko.md | 9 KO |
| Flow Atlas | 04_flow-atlas/flows.md | 7 类流 |
| Candidates | 05_candidates/candidates.md | 7 项 |
| Validation | 06_validation/ | 3 报告 |

## 关键数字（来自仓库事实）

- commit `ccaf9fc0ffe6ac39c7ec786af7608ab1de19467b`（[skip ci] Release new versions，2026-09-18）
- js-sdk / python-sdk 版本 2.51.0；pnpm@10.34.5 monorepo
- 测试 169 个 `*.test.ts`（vitest，live sandbox 60s 预算）
- 默认参数：requestTimeout 60s / retries 3 / sandbox 300s / keepalive ping 50s / 连接 100 / inflight 1000
- envd 端口 49983（REST）/ Jupyter 端口（code-interpreter）
- 签名 URL 前缀 `v1_`；IAM 占位符 `${e2b.identity.tokens.<name>}`；Secret 占位符 `${e2b.secrets.<name>}`

## 质量指标速览

| 指标 | 值 |
|---|---|
| EK 数 | 32 |
| EK 平均出边数 | 2.2（≥1 合格） |
| 游离 EK 比例 | 0% |
| 聚合规则覆盖率 | 100%（9/9 KO） |
| KO 平均簇规模 | 3.56（3~12 健康） |
| 盲重建 | performed（Truth + Coverage + 独立 Auditor） |
| 独立验证 | 20 CONFIRMED / 4 PARTIAL / 1 OVERGEN / 3 MISSING / 1 CONTRADICTED → 已 Reconciliation |
