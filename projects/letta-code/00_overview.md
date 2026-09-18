# 00 Overview — letta-code（Letta）

> run: ARCH-2026-09-18-001 · mode: initial · commit `3df2ebd6` · version 0.32.12 · skill v3.2

## 项目定位
Letta（前身 MemGPT）的**当前源码仓库**——一个"memory-first"的 stateful agent harness。核心命题：**代理应该像人一样有持续的身份、记忆和学习能力**，而不是无状态地每次从零开始。README 明示 `letta-ai/letta`（24.7K★）main 分支已归档为文档仓，**当前源码全部在 `letta-code`（3.4K★，TypeScript+Bun，Apache-2.0）**；`archive` 分支保留旧 V1 API 服务端。

## 本次考古回答的核心问题
1. "记忆像人一样"是如何实现的？→ **记忆 = 每个 agent 一个本地 git 仓库 + Markdown 文件树（MemFS）**，系统提示词每次编译时把已提交的核心记忆文件投影进上下文（`{CORE_MEMORY}` 注入），agent 通过 memory 工具自编辑记忆，每次写操作经 git commit 留痕，turn 后 push 到远端 `$LETTA_MEMFS_BASE_URL/v1/git/$AGENT_ID/state.git`。
2. 记忆写入-检索-演化回路是什么？→ 写入（memory 工具 6 命令 + memory_apply_patch）→ pre-commit hook 校验 frontmatter → commit → post-turn push；检索（系统提示词投影 core memory + 文件延迟读取）；演化（reflection/history-analyzer/memory/init 四类 memory-subagent 子代理 + reflection settings 触发）。
3. harness 治理面如何构成？→ **四层**：PermissionMode（unrestricted/standard/acceptEdits/strict）、内核文件系统沙箱（seatbelt/bwrap，fail-closed）、cross-agent 记忆隔离（deniedRoots 墙）、memory-confinement（记忆子代理默认强制内核沙箱）。另加 Mods（能力插件）、Skills（程序性知识）、AI_POLICY（贡献治理）。
4. 09-18 雷达假说验证：**"记忆 = Markdown 仓库"路线在 Letta 中是生产级实现**——git-backed MemFS + frontmatter + pre-commit 校验 + 系统提示词投影，与 OKF/Grok Build/Anthropic /mnt/memory 的"记忆=文件"路线同构，且是其中最工程化的实现之一。

## 三层知识配比（Reconciliation 后）
| 层 | 数量 | 说明 |
|---|---|---|
| Facts / Evidence | 121 | 见 01_project-layer 与 02 EK 的 evidence 引用 |
| Engineering Knowledge | 49（≥80 条边，avg 1.63） | 见 02_engineering-knowledge/ek-graph.md |
| Generalized KO | 9（L3×6 / L4×2 / L5×1） | 见 03_knowledge-layer/ |
| Candidates | 8（2 项经独立审计修正） | 见 05_candidates/ |
| Flow Atlas | 7 类流 | 见 04_flow-atlas/ |
| Validation | 31 CONFIRMED / 9 PARTIAL / 2 DOWN / 3 OVERGEN / 0 CONTRADICTED / 1 REVIEW | 见 06_validation/ |

## 关键发现（顶部摘要）
- **KO-01「记忆即文件系统」**：MemFS 把"记忆"从抽象概念降为**可审计的物理资产**——git 历史 = 记忆演化史；frontmatter = 记忆元数据；pre-commit hook = 记忆写入门禁。
- **KO-02「fail-closed 治理」**：无内核沙箱可用时 memory-confinement 直接抛错（不降级弱策略）；记忆子代理默认强制沙箱（无 approve/deny 可回退），与交互代理 opt-in 沙箱形成对照。
- **KO-03「记忆权限模型 = 路径模型」**：跨 agent 隔离不是"权限列表"而是**文件系统路径墙**（`~/.letta/agents` denied），自我记忆通过路径 carve 回写——隔离边界与沙箱策略同源（FsSandboxPolicy 顺序语义）。
- **KO-04「自编辑记忆 + 提交理由」**：每次记忆写操作强制 `reason`，agent 必须为"为什么改记忆"留下解释——记忆变更成为可追溯的决策记录。
- **09-18 假说**：雷达"记忆=Markdown 仓库路线成型"得到**强支持**（生产级实现 + 双格式演进 v1→v2 + 迁移 skill 存在）。

## 主要争议（供审计）
- **D1**：v2 中 `read_only` 字段被移除（仅 legacy 生效）——"只读保护"在 v2 中被 frontmatter 白名单（name/description）取代，是设计演进还是安全降级？Auditor 判 PARTIALLY_CONFIRMED。
- **D2**：`DEFAULT_PERMISSION_MODE = "unrestricted"` 与"治理严苛"叙事张力——fail-closed 集中在记忆子代理与沙箱路径，交互代理默认权限却是最宽松档。
- **D3**：记忆 = git 仓库的远程同步依赖 `api.letta.com` fallback——本地后端如何避免把代理 URL 持久化进 git config（memfs-git-proxy 设计，Reconciliation 后 EK-48 已完整记录）是隐式契约。
- **D4（独立审计新增，NEEDS_HUMAN_REVIEW）**：mods 三源全部 `trusted:true` + 主进程动态加载（无沙箱）——与记忆子代理 fail-closed 形成对比，需 Owner 确认是有意设计（可信插件模型）还是待加固面。

## 产物清单
| 文件 | 内容 |
|---|---|
| 01_project-layer/project-map.md | 项目地图（架构/模块/生命周期/依赖/入口） |
| 01_project-layer/project-facts.md | 121 条 L0/L1 事实（带证据） |
| 02_engineering-knowledge/ek-graph.md | 49 条 EK + 六类边（links）+ 证据（含 Reconciliation 新增 EK-47/48/49） |
| 03_knowledge-layer/ko.md | 9 个 KO（aggregation_rule R1-R4 + 升维论证） |
| 04_flow-atlas/flows.md | 七类流（Control/State/Data/Evidence/Authority/Memory/Policy） |
| 05_candidates/candidates.md | 8 个未验证假设（C-02/C-05/C-08 经独立审计修正） |
| 06_validation/validation.md | Truth/Coverage/Causality/Flow/Abstraction/Counterexample/Epistemic 报告 |
| 06_validation/independent-audit.md | 独立 Auditor 盲重建报告（阶段 5 产出） |
| 06_validation/reconciliation.md | Reconciliation 修正清单（阶段 6，10 项） |
| run_metadata.yaml | run 元数据（run_id/repository/commit/skill_version/metrics/validation） |
