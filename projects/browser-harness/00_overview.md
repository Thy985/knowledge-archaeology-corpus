# 00 Overview — browser-harness（ARCH-2026-10-04-001）

## 一句话定位
**自愈浏览器 harness**：browser-use 团队推出的极简 CDP 连接层——让 LLM（agent）经一根可编辑 WebSocket 直连真实 Chrome，agent 在执行中自写缺失 helper（agent-workspace/agent_helpers.py 运行时注入），"harness 每次运行自我改进"。README 原话："The agent writes missing helpers as it works, so the harness improves with every task."

## 考古基本信息
| 字段 | 值 |
|---|---|
| run_id | ARCH-2026-10-04-001 |
| repository | browser-use/browser-harness |
| commit | afbcc381b963040c19627d788e40c7e7663171ee（2026-09-07，Merge PR #757） |
| skill_version | knowledge-archaeology v3.2（生产版） |
| mode | initial |
| src 规模 | 6,732 行 / 14 模块；tests 4,099 行 / 13 文件 |
| 三层配比 | Facts 100+（见 01/02 证据引用）→ EK 45 → KO 8 |

## 核心认知（三层纵向链预览，详见 03）
```
Engineering Fact（helpers.py 底部 _load_agent_helpers() 把 agent-workspace/agent_helpers.py 注入 globals —— L1）
    ↓
Engineering Knowledge（核心代码受保护，agent 只编辑 agent-workspace；daemon 持有 CDP 会话权威，agent 只持有任务 —— L2）
    ↓
Pattern（"受控中间层 + 运行时能力扩展"：守护进程持有会话/权限权威，agent 在沙箱工作区自扩展能力 —— L3）
    ↓
Cognitive Model（Agent 对真实系统的控制 = 权限契约：会话权威与任务能力分离，能力可运行时增长而权威不可 —— L4，cross-project pending）
    ↓
Methodology（Agent 影响真实外部状态的动作必须经守护中间层；agent 自扩展能力须限定在独立工作区 —— L5，cross-project pending）
```

## 结构
- 01_project_layer.md — 项目地图（浓缩自 snapshot_artifact.md）
- 02_engineering_knowledge.md — 45 EK（EK Graph，links 六类边）
- 03_knowledge_layer.md — 8 KO（R1-R4 聚合规则）+ L3/L4/L5 分层
- 04_flow_atlas.md — 七类流（Control/State/Data/Evidence/Authority/Memory/Policy）
- 05_candidates.md — 未验证假说（含 SKILL.md name 冲突等）
- 06_validation.md — 盲重建验证 + 反例 + 质量指标
- run_metadata.yaml — run 元数据
