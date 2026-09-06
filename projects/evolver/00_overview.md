# Evolver — Project Archaeology Overview

- **run_id**: ARCH-2026-09-06-002
- **project**: evolver
- **repository**: https://github.com/EvoMap/evolver.git
- **commit**: 31b0691（main，"feat(skill-store): report Hub install-success after local fetch commit (#622)"）
- **skill_version**: knowledge-archaeology v3.2
- **timestamp**: 2026-09-06（UTC+8）
- **mode**: initial

## 一句话定位

**Evolver 是一个 GEP（Genome Evolution Protocol）驱动的 AI Agent 自进化引擎**（EvoMap 出品，v1.94.0，9k stars，arXiv 2604.15097）：把 agent 的运行日志/错误模式/信号扫描出来，匹配本地基因/胶囊资产库，生成"协议约束的进化 prompt"（Evolver 自称 prompt generator 而非 code patcher），并经 solidify 验证补丁、canary 金丝雀、rollback 回滚形成完整闭环。配套 EvoMap Hub（A2A/ATP 协议）构成"进化网络"。

## 为什么值得考古（Knowledge Value）

1. **Agent/AI Engineering 相关性（S 级）**：这是"自进化引擎"的实现者——**会修改自身源代码的系统**（EVOLVE_ALLOW_SELF_MODIFY / solidify 修改 src/**）。自我修改、自我更新（forceUpdate）、自我修复（distiller 蒸馏修复 gene）三自一体的工程样本极其罕见。
2. **Novelty**：①**源码混淆**——57/202 个 src 文件被 javascript-obfuscator 混淆（evolve 全模块 + gep 半数），README 声明"转向 source-available"，GPL-3.0-or-later 声称 vs 混淆现实 ②**测试即静态分析**——fetchSecurity 测试直接 grep index.js 源码模式做 GHSA 回归 ③**资产永不覆盖承诺**——forceUpdate 对用户 GEP 资产（genes/capsules/events）的升级保护。
3. **Benchmark 学习价值**：①doc-impl 矛盾活案例（README 说 validation 白名单 node/npm/npx，实现只有 node）②安全纵深分层（sandboxExecutor 白名单 + BLOCKED_NODE_FLAGS + 元字符拒绝 + 180s 超时 + cwd 限定）③conformance golden-vectors（token 节省公式精确匹配）。
4. **生态相关**：KnowlegeMap 雷达的 hermes-agent（OpenClaw）与 Evolver 的"相似性分析"争议（README Notice 声明）——与用户雷达三卡增量直接连接。

## 认知核心（一句话）

**"自进化"系统的可信性 = 协议约束 + 可审计资产 + 纵深安全 + 自我更新可恢复**：Evolver 用 GEP 协议约束进化（不 improvisation）、用本地资产 + 事件日志保证可审计、用沙箱/白名单/路径遏制保证安全、用备份/日志/原子重命名/金丝雀保证自我更新可恢复——而这一切建立在"核心逻辑混淆、文档声明与实现存在差距"的复杂治理现实之上。

## 关键数字（均可回溯）

| 指标 | 值 | 来源 |
|------|-----|------|
| 规模 | 470 文件；src/ 202 js / 36,200 行 | find+wc |
| 版本 | v1.94.0（无 git tag 于 --depth 1 快照，package.json） | package.json |
| 许可证 | GPL-3.0-or-later（2026-04-09 起；前为 MIT） | package.json+README |
| 混淆 | **57/202 src 文件混淆**（evolve 8/8、gep 44/86、proxy 4/29） | grep _0x |
| 测试 | **221 个 test 文件**（node --test） | ls test/*.test.js |
| 核心入口 | index.js（186KB CLI，可读）+ src/evolve.js（混淆） | wc |
| 依赖 | @evomap/gep-sdk / @evomap/atp-sdk / @aws-sdk/client-bedrock-runtime / undici / eventsource / dotenv | package.json |
| Node | >= 22.12 | package.json engines |
| arXiv | 2604.15097（4590 对照试验，45 科学编码场景） | README |

## 产物清单

- `01_project-layer/` — 项目地图（六面架构 + 混淆状态图 + 数据结构）
- `02_engineering-knowledge/` — EK Graph（30 条 EK + 6 类边）
- `03_knowledge-layer/` — Generalized KO（7 KO + CM + M）
- `04_flow-atlas/` — 七类流
- `05_candidates/` — 未确认假说（混淆/GPL、doc-impl、self-modify 语义）
- `06_validation/` — Validation 报告 + 反例 + 质量指标
- `run_metadata.yaml` — run 元数据

## 免责声明

本包事实均可回溯 31b0691 快照。**盲区声明**：src/evolve.js 与 src/evolve/pipeline/*（8 文件）及 gep 44 文件被混淆，内部逻辑不可读——相关结论均以 index.js（可读 CLI）调用点、README 文档、test 文件（可执行契约）为证据，诚实标注为"外部观察"而非"内部实现精读"。抽样测试 60 pass / 2 fail（fail 因 @evomap/gep-sdk 未安装，非回归）。
