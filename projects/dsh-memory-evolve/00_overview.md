# dsh-memory-evolve — Project Archaeology Overview

- **run_id**: ARCH-2026-09-06-003
- **project**: dsh-memory-evolve
- **repository**: https://github.com/csyangwen/dsh-memory-evolve.git
- **commit**: 999a0edb39894ebb2c5ceeb97712db8c3a9c5093（main）
- **skill_version**: knowledge-archaeology v3.2
- **timestamp**: 2026-09-06（UTC+8）
- **mode**: initial

## 一句话定位

**dsh-memory-evolve 是 DeepSeek Harness（DSH）的长期记忆 + 自我进化插件**（v0.1.0，MIT，零运行时依赖）：为 DSH 的 AI 带来跨会话五轨记忆（用户档案/全局事实/项目关键记忆/项目日志/每日日志）、双轨道写入（用户确认轨 vs 学习建议轨）、回合内自我记忆审查、技能自动创建与治理、四轨待办、跨设备记忆同步（git 分支 + 三路合并）、会话编排与广播（COI）、外部 AI CLI 统一派单（kimi/codex/grok/hermes）、会话隐形评审员（advisor）与 WebUI 管理界面。

## 为什么值得考古（Knowledge Value）

1. **Agent/AI Engineering 相关性（S 级）**：这是"AI 长期记忆系统"的完整实现样本——记忆的分层（五轨）、写入的权威链（用户确认 vs 建议）、记忆的漂移防护（round-trip 守卫）、记忆的同步（git 分支 + 三路合并）全部有可读实现（lib/ 50k 行 JS 未混淆）。
2. **Ecosystem Connection（首个生态连接案例）**：corpus 已有 deepseek-harness 考古结果——本插件是 DSH 生态的正式插件（`dsh plugin add` 标准安装，cordis.patch.yml bundle 自动注册），与 Evolver 考古中发现的 hermes 派单技能（skills/hermes-cli-calling）形成 KnowlegeMap 雷达 hermes-agent 的连接点。**同一生态的双项目考古，是跨项目验证的起点。**
3. **Novelty**：①回合内记忆审查"主 LLM 自审"（不派 subagent，插件只做节拍器，且计数器**不自动重置**——防静默丢弃）②三路合并器"git 冲突标记永不落盘"③跨协议 URL 归一化身份（https/ssh 收敛同键，"双设备认亲"）④ws-coord 软模式（先信任 AI，检测到冲突只警告不拦截，但 fs/observed 自动登记是硬证据）⑤发射闸门 NFKC 归一化 + 空泛短语抑制。
4. **Benchmark 学习价值**：①测试环境隐式契约三件套（git 默认分支 main / git 身份 / darwin 平台假设）——798 断言在默认环境 48 挂、配置后 3 挂（全为 darwin），这是"环境契约 vs 真实回归"的活教材 ②60 个测试文件主题化命名（advisor-*/sync-*/coi-*/review-*）。

## 认知核心（一句话）

**"AI 的记忆可信性 = 用户确认 + 可审计轨道 + 漂移防护"**：Evolve 用五轨分层让记忆可检索、用双轨道（用户确认 vs 建议确认）把权威交给用户、用漂移守卫/原子写/跨进程锁保护本地文件、用 git 分支 + 三路合并实现跨设备一致——而这一切通过"纯插件零侵入"（只走 DSH public seams）交付。

## 关键数字（均可回溯）

| 指标 | 值 | 来源 |
|------|-----|------|
| 规模 | lib/ 50,144 行 JS（后端权威实现）+ src/ 21,737 行 TS/TSX（前端）+ 202 文件 | wc/find |
| 版本 | 0.1.0 | package.json |
| 许可证 | MIT | package.json |
| 依赖 | **零运行时依赖**（node:fs only） | lib/*.js 头部注释 |
| 测试 | 60 文件 / 798 断言（node --test） | tests/*.test.js |
| 实测 | 干净环境配置后 **795 pass / 3 fail（全为 darwin 平台假设）** | 本 run 实测 |
| 构建 | scripts/build.mjs（esbuild 从 DSH checkout 解析） | build.mjs |
| 注入 | cordis.patch.yml（bundle 自动注册 host 插件行） | cordis.patch.yml |
| 外部派单 | skills/{codex,grok,hermes,kimi}-cli-calling | skills/ |

## 产物清单

- `01_project-layer/` — 项目地图（双轨架构 + 模块面 + 数据结构 + 治理）
- `02_engineering-knowledge/` — EK Graph（28 条 + 6 类边）
- `03_knowledge-layer/` — Generalized KO（8 KO + CM + M）
- `04_flow-atlas/` — 七类流
- `05_candidates/` — 未确认假说
- `06_validation/` — Validation 报告 + 反例 + 质量指标
- `run_metadata.yaml` — run 元数据

## 免责声明

本包事实均可回溯 999a0ed 快照。lib/ 为权威执行面（测试直接 import），全部可读无混淆。抽样/全量测试 798 断言实测（3 fail 为 darwin 平台假设，非回归）。
