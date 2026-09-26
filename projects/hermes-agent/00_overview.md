# 00 Overview — Project Archaeology Package（hermes-agent）

- **run_id**: ARCH-2026-09-19-001 ｜ **repository**: https://github.com/NousResearch/hermes-agent.git
- **commit**: `fba4cb1`（2026-09-18, main, v0.21.3）｜ **mode**: initial ｜ **priority**: A
- **一句话定位**: Hermes Agent 是 Nous Research 的个人 AI Agent——同一 agent 核心服务 CLI/消息网关(~20 平台)/TUI/Desktop，跨会话学习（记忆+技能），用"冻结快照记忆 + 程序性技能记忆 + OS 级唯一安全边界"构建成长型 harness。
- **考古核心命题**: 这个项目让我们认识到——**长生命周期 agent 如何在不破坏 prompt cache 的前提下实现记忆持久化与自我改进？**

## 三层知识速览
| 层 | 数量 | 代表 |
|---|---|---|
| Project Facts | 35 | 记忆冻结快照 / skills_guard v5 / SQLite WAL 族 / SECURITY §2.2 |
| Engineering Knowledge | 41（EK Graph） | 冻结快照契约 / 供应链信任分级 / WAL 兼容回退 / 回滚权威模型 / Curator 不变量 |
| Generalized KO | 9（L2×2/L3×4/L4×3） | KO-02 唯一承重边界原则 / KO-01 记忆冻结契约 / KO-06 自我改进闭环 |
| Candidates | 9 | C-01 冻结 vs 即时生效谱系 / C-03 供应链防线三层光谱 |
| Flow Atlas | 7 类 | Control/State/Data/Evidence/Authority/Memory/Policy |

## 最尖锐的跨项目对照（vs letta-code ARCH-2026-09-18-001）
1. **安全哲学**：hermes 明文"OS 隔离是唯一承重边界，in-process 防御非边界"（SECURITY.md §2.2）vs letta 的 fail-closed 内核沙箱（seatbelt/bwrap）——两项目对"承重墙"定义相反，但都拒绝把启发式当边界。
2. **记忆写入时机**：hermes 冻结快照（session 粒度一致性，cache 优先）vs letta post-turn push（每轮后提交）——同一谱系两端。
3. **记忆载体**：SQLite WAL 状态库 vs git-backed MemFS 文件树——"本地优先 agent 的持久层"两种完整实现。

## 本 run 关键发现
- prompt cache 神圣性作为**第一设计约束**反向塑造了记忆/技能/工具三系统的形态（冻结、deferred、narrow waist）。
- 供应链安全走"信任分级 + 静态扫描 + 诚实声明缺口"路线（skills-guard-v5），与形式化验证（SkillFortify）互补。
- 状态韧性工程族（WAL 回退/生产测试隔离/回滚权威/checkpoint shadow git）是 SQLite 本地优先应用的可迁移范本。

## 关键争议
- user_authored 注入例外（#112570）的安全权衡（C-07）；单外部 provider 约束的长期扩展性（C-08）——均留 Candidates。
- 记忆 provider 的 fail-closed checkpoint（v2）与 best-effort（v1）混跑语义。

## 产物清单
00_overview.md ｜ 01_project-layer/project-facts.md ｜ 02_engineering-knowledge/ek-graph.md ｜ 03_knowledge-layer/ko.md ｜ 04_flow-atlas/flows.md ｜ 05_candidates/candidates.md ｜ 06_validation/validation.md ｜ run_metadata.yaml
