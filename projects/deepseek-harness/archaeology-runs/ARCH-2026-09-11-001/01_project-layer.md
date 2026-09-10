# 01 — Project Layer（v0.1.5-rc.2 增量地图）

> 完整项目地图见 09-03 考古包 01_project-layer/。本节只记 **v0.1.2→v0.1.5 的增量与 09-03 遗漏项**，全部可回溯仓库实际内容。

## 1.1 仓库结构变化（v0.1.5）

| 位置 | 变化 | 证据 |
|---|---|---|
| `python/sdk` + `python/sdk-runtime` | Python SDK 成型（pyproject/uv.lock/7 测试文件/111 tests） | python/sdk 目录；pytest 实测 |
| `packages/api/workspace-files` | 新增 workspace 文件服务包（Host `ctx.workspaceFiles` + Client `file` provider） | 2026-09-05-workspace-files-service.md |
| `packages/feedback` | 规范化反馈日志（canonical-feedback-log + nonofficial-feedback-otel） | notes 09-05 |
| `.agents/notes/` | 862→958 条（+96，09-05 后加速） | git ls-tree 实测 |
| `SAFETY.md` / `BENCHMARK.md` | 09-03 已存在但考古未引用；本轮纳入 | 文件实测 |
| 顶层 | `apps/`（web/electron）、`website/`、`vendor/`、`native/`、`snapshots/` | ls 实测 |

## 1.2 核心模块（v0.1.5 视角）

```
packages/core/session      # Session surface（V3 envelope 校验所有权）
packages/api/workspace-files # Host 服务 + Remote namespace + Client provider（v0.1.5）
packages/sandbox+guard     # 权限：sandbox 三态 / approval / guard 单调拒绝
packages/code-runtime      # 执行世界：PTC run_code / python fd3 协议 / e2b 适配
packages/llm               # provider-routed LLM adapters
packages/subagent          # 子代理委托（policy 继承 / model-selected routes）
packages/acp               # ACP automation-only 协议桥（09-03 coverage_gap 补）
packages/feedback          # 反馈日志 + OTEL（v0.1.5）
python/sdk                 # subprocess JSON-RPC over stdio（v0.1.5）
```

## 1.3 生命周期（新增/变化）

- **会话格式 V3**（2026-09-06 canonical session envelopes）：`system/message`、`user/message`、`assistant/message`、`tool/result` 必须携带 `surfaceOp`（`'append'` 或 `{op:'replace',startSeq,endSeq}`）；替换端点在当前 surface 序，非数值序；接受规则校验成员资格/有序端点/引用覆盖/仅内容单节点替换。冲突的 tool 失败元数据**不推断**错误结局。
- **Python SDK 生命周期**：`pip install deepseek-harness-sdk` → 启动捆绑 `dsh` CLI（`--profile sdk`）→ 显式 `dsh_home`（**绝不**发现 `~/.dsh`）→ 新行分隔 JSON-RPC over stdio。

## 1.4 配置（增量）

| 配置 | 值 | 证据 |
|---|---|---|
| `tools.mode` | `'code'` → `'ptc'`（无兼容别名） | 2026-08-25-rename note |
| presets 目录 | `presets/code/` → `presets/ptc/`（preset id `ptc`） | 同前 |
| SDK profile | `--profile sdk` + 显式 `DSH_HOME` | python/sdk README |

## 1.5 权限与治理（增量）

- **workspace-file-read-authority**（09-09）：read 族继承 fs 后端授权；list/changes workspace-scoped；readRelated 允许 `..`；Document Preview iframe `sandbox="allow-scripts"`（opaque origin 防父访问，保留网络）——**有意取舍**。
- **PTC executor collapse**（08-07）：collapsed call 在 `createExecution`（策略管线之前）确定性拒绝。
- 09-03 已覆盖的 approval/sandbox/guard 机制无变化（本轮未发现反证）。

## 1.6 外部依赖（增量）

- Python SDK：pydantic>=2.12、deepseek-harness-runtime-bin（同版本 wheel）、hatchling 构建、uv.lock。
- Node/CI 治理 notes：node-engine-floor、python-runtime-windows-hosted、preview-hosted-runner-sizing（process/*，09-06）。
