# 00 · Overview — ThinkingBox（microsoft/thinkingbox）项目考古总览

- **Run ID**: ARCH-2026-10-08-001
- **目标仓库**: https://github.com/microsoft/thinkingbox.git @ `892964e7226044e5463ad188da119df838f4bec1`（main，v0.1.0，MIT）
- **考古引擎**: knowledge-archaeology v3.2（本地生产版）
- **定位一句话**: ThinkingBox 是一个面向**有状态业务工作流**的 Agent 评测沙箱与基准框架——不测"单次答对"，而测"多次连续执行都稳定正确"（pass^k），评分真相来自 MCP server 的**世界状态（effects）**而非 LLM 自述。

## 1. 这个项目让我们认识到了什么（核心命题）

Agent 评测最大的幻觉是 **pass@1**：模型试一次成功，不代表在有状态、多轮、可恢复的业务工作流中可靠。ThinkingBox 用三层机制对抗这种幻觉：

1. **状态真相评分**：评测不看模型说什么，而看 MCP server 的 `effects`（世界状态取证）——agent 说"文件已上传"不算数，server 的 files 列表变了才算数。
2. **pass^k 统计**：每个测试用例**连续执行 k 次**，全部通过才算过（`(c/n)^k`）；配合 **goldilocks zone**（Beta 后验 P(0.0625<p<0.9375|data)≥0.95）拒绝小样本下的"看起来稳定"。
3. **可复现评测管道**：agent 解码与测试执行解耦（可重跑测试不动 agent）、错误行重试（previous-results）、每次 run 落 run_metadata.yaml（agent/user/setup 配置全记录）。

## 2. 项目边界（三仓库拆分）

| 仓库 | 内容 | 本仓考古范围 |
|---|---|---|
| github.com/microsoft/thinkingbox | **评测框架本身**（本次考古对象） | ✅ |
| github.com/microsoft/thinkingbox-data | 真实语料 + servers + 支持数据（**独立仓库**） | ❌ 未考古 |
| github.com/microsoft/thinkingbox-training | RL 后训练（GRPO/LoRA/多轮 MCP rollout/FSDP2） | ❌ 未考古 |

**外部宣称边界（重要）**：论文（arXiv:2608.19741 "One Success Isn't Reliability: ThinkingBox, a Sandbox and Benchmark for Agents in Stateful Business Workflows"）与新闻中引用的统计数字——121,680 trials × 12 模型、79,853 次未过可执行检查、67.24% 失败仍干净结束无报错、79.9% 失败源于工具处理、每任务 20 次连续执行、pass@1 幻觉终结——**全部来自 thinkingbox-data 语料，本仓库只是评测框架，这些数字在本 run 中只能标记为外部宣称（provenance=论文/新闻），不得当仓库内可验证 Fact**。本仓内可复现的是"按 effects 状态真相评分 + pass^k 指标"的执行机制。

## 3. 仓库事实基线（可复现）

- commits：18（全在 2026-10，浅克隆 depth=50 拉全）；HEAD `892964e`（"Update README with training repository details (#38)"，2026-10-03）
- 包内 Python 文件：55 个；核心源码 12,832 行（`thinkingbox/**/*.py`）；全仓 168 files
- requires-python >= 3.10；无 tag（v0.1.0 来自 pyproject）
- CLI：`tb` = `thinkingbox.cli.main:main`（click）；子命令 `agg / dump-tests / infer / mcp-start / pp / run-test / sbs / tui`
- 测试：`tests/` 34 py；`tb mcp-start --servers tests/servers.yaml` + `pytest -v tests`；marker `typesense`
- 治理证据（git log）：pin SHA（#29 ea053ba）、token 权限最小化（#19 126808f）、CodeQL（#20 94f2bda）、Scorecard（#15/#16）、pre-commit black/isort/flake8、SECURITY.md 存在

## 4. 三层知识架构速览（宽底座 + 窄尖顶）

```
                L4 Cognitive Models (2)          ▲ 窄尖顶
               / L3 Patterns (14 条证据内)       │ 9 Core KO（R1-R4 聚合）
              /───────────────────────\         │
             / L2 Engineering Knowledge          │ 宽底座
            /  55 EK（EK Graph，6 类边）         │ 推理原材料（无损）
           /───────────────────────────\        │
          / L1 Facts（100+，内联证据）          │
         Repository Evidence（18 模块源码）      ▼
```

- **Project Layer**（01）＝ 项目地图：CLI 入口/模块/核心数据结构/生命周期/配置/治理
- **Engineering Knowledge**（02）＝ 55 条 EK 图：执行链/循环/代理/统计/水合/夹具/配置
- **Knowledge Layer**（03）＝ 9 个 KO：评测真相性、双层隔离、注入防护、统计防幻觉、防失控循环、会话生命周期、副作用安全、可复现审计
- **Flow Atlas**（04）＝ 七类流（Control/State/Data/Evidence/Authority/Memory/Policy）
- **Candidates**（05）＝ 外部宣称 + 跨项目假说 + 待验证模式
- **Validation**（06）＝ Blind Reconstruction 报告

## 5. 关键发现（最高信号）

1. **评测真相 = 世界状态，不是 LLM 输出**。`__reserved__geteffects` 协议让每个 MCP server 自报状态；`effects` 双源（server 自报 + proxy 调用日志 `__reserved__proxy_info.tool_calls`）互证。
2. **"工具副作用安全"是硬约束**：`max_retries_timeout=0`——MCP 调用超时**绝不重试**（"on timeout, the operation might have happened on the server. Do not re-try."），防重复副作用；失败工具通过 `tc.metadata["error"]` 短路，不重调。
3. **统计层刻意对抗小样本自信**：pass^k 用**有意 biased** 的 `(c/n)^k`（unbiased 在 c<k 时为零、无区分度）；goldilocks zone 用 Beta 后验判定"稳定通过"。
4. **测试在子进程隔离执行**（`python -m thinkingbox.cli.testscript_worker`），框架内 `exec` 无隔离（代码注释直言 "NO ISOLATION HERE!"）；stdout 用 `\0` 分隔符定位响应。
5. **用户模拟器防幻觉**：COPY-ONLY 实体规则（ID/数字必须从 USER_CONTEXT 逐字复制），SANITIZE 防 prompt 结构混淆。
6. **部署形态务实**：`tag_taxonomy.yaml`（gitignored）注入 Microsoft 内部标签体系，公开仓回退 `tag_taxonomy.example.yaml`——分类法按部署注入而非硬编码。

## 6. 与本知识库既有 Corpus 的连接

- **agentevals / deepseek-harness**：同为 agent 评测执行器（ThinkingBox 的 effects 取证 vs deepseek-harness 的痕迹断言——对比见 05_candidates）
- **mcp 工具线**（mcp/ 考古）：ThinkingBox 的 MCP server 协议（`__reserved__init/geteffects/teardown`）是 MCP 之上的评测扩展约定
- **owasp-agentic-skills**：注入防护思路（untrusted 指令 + 编码）同族
- **eval 统计线**（agentevals）：pass@k 无偏估计与 goldilocks 贝叶斯判定是 agentevals 口径的升级版

## 7. 考古范围声明

- **深读**：18 个 Python 模块全量（见 run_metadata.scope），README 520 行、pyproject、git log 全量
- **抽样**：`tests/`（34 py 未逐行读，抽样 servers.yaml 等）、`docs/`（行为已从代码反推，关键文档未逐篇读）、`tools/toolslib/cloud_drive.py`（CloudDrive 实现细节未逐行读，但 effects 列表存在已由 mcp_cloud_drive.py 引用确认）
- **外部边界**：thinkingbox-data / thinkingbox-training 不在本仓，其内容不产生本仓内 Fact
