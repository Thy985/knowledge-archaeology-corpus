# dsh-memory-evolve — Knowledge Archaeology Index

**一句话定位**：DeepSeek Harness（DSH）的长期记忆 + 自我进化插件（v0.1.0，MIT，零运行时依赖）——五轨记忆（用户档案/全局事实/项目关键记忆/项目日志/每日日志）、双轨道写入（用户确认 vs 学习建议）、回合内自我审查、技能治理、跨设备 git 同步、会话编排与外部 AI CLI 派单（kimi/codex/grok/hermes）。

**考古日期**：2026-09-06 ｜ **run_id**：ARCH-2026-09-06-003 ｜ **commit**：999a0ed（main） ｜ **skill**：knowledge-archaeology v3.2 ｜ **mode**：initial

## 产物清单

| 层 | 文件 | 内容 |
|----|------|------|
| Overview | 00_overview.md | 定位/价值/认知核心/生态连接 |
| Project Layer | 01_project-layer.md | 双轨架构 + 12 模块面 + 数据结构 + 治理 |
| Engineering Knowledge | 02_engineering-knowledge.md | 30 EK（EK Graph，6 类边，100% links，全部 Fact 级） |
| Knowledge Layer | 03_knowledge-layer.md | 8 KO（R1-R4）+ 4 CM + 4 M |
| Flow Atlas | 04_flow-atlas.md | 七类流（全部可回溯——无混淆盲区） |
| Candidates | 05_candidates.md | 8 候选（2 跨项目假说） |
| Validation | 06_validation.md | 反例 14 次 + 实测 798 断言（795 pass/3 darwin 假设） |
| Run 快照 | archaeology-runs/ARCH-2026-09-06-003/ | run_metadata.yaml + 全部层副本 |

## 关键发现（摘要）

1. **双轨记忆 + 确认门**：user track（显式/确认）vs learned track（建议待确认）；审查节拍器不自动重置（防静默丢弃）（KO-01）
2. **文件写入安全三件套**：Hermes 兼容格式 + 漂移守卫（round-trip 重写验证）+ 跨进程锁/原子写（KO-02）
3. **跨设备同步链**：URL 归一化身份（https/ssh 认亲）→ 三路合并（冲突标记永不落盘）→ decideModeB（绝不碰 main）（KO-03）
4. **AI 动作治理**：read-before-write 证据门 + kebab-case 规则化 + 评审发射闸门（NFKC 归一化/空泛抑制/升级放行）（KO-04）

## 生态连接（首个同生态双项目考古）

- **deepseek-harness**（corpus 已有）→ 本插件是 DSH 正式插件（cordis.patch.yml bundle 注册）
- **evolver**（同日考古）→ skills/hermes-cli-calling 派单技能（EvoMap 生态）
- **KnowlegeMap 雷达** → hermes-agent（OpenClaw 争议）与两项目形成"外部 AI 统一调度"汇合点

## 关键争议

- 无 NEEDS_HUMAN_REVIEW（lib/ 全部可读，无混淆盲区——与同日 evolver 考古形成可信度对照）
- C-07：测试环境契约未文档化（git main 分支/身份/darwin 假设）——本 run 实测 48 挂→配置后 3 挂

## 实测记录

全量 798 断言：默认环境 750 pass / 48 fail → 配置 git 契约后 **795 pass / 3 fail（全为 darwin 平台假设，非回归）**。
