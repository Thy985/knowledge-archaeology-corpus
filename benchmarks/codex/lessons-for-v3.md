# Lessons for v3 —— 三层知识架构升级蓝图

> 由 Codex v2 考古 + 用户方法论升级驱动。本文件是 knowledge-archaeology v2 → v3 的设计依据，保留为 skill 演进的证据。

## 触发：用户指出的结构性问题

用户审查 Codex 知识库后指出：

1. **知识库不能只有"升维后的知识"**——原始事实、关键实现、失败案例、决策过程同样是认知资产，只是层级和用途不同
2. **当前结构存在"过度压缩"**：项目代码 → 大量工程事实 → 抽象 → 只保留 7 个 KO。底层信息被视为不重要
3. **"不够抽象"绝不等于"不重要"**：如 `Weak<Session> → session 释放 → ask("not_allowed")` 回答"控制面生命周期结束时代理层怎么办"，设计 API Gateway / Sidecar / Callback Runtime 时完全可复用

## 核心洞察

- 高层 Knowledge = 压缩后的可检索认知；底层 Engineering Knowledge = 推理所需的无损底座
- 未来 Agent 遇到新问题（如"如何设计不会悬挂回调的网络审批代理"）需要的是**底层事实**（Weak<Session> / ActiveNetworkApproval / cancellation_token / ask("not_allowed")），据此重新推理出模式——不是直接查一个 L4 结论
- **抽象错了 → 可回到底层重新推导**；只保存 KO → KO 错了甚至不知道它是怎么得出来的

## v3 设计需求（F7-F11）

| 编号 | 需求 | 落地 |
|------|------|------|
| F7 | 三层知识架构（Project / Engineering / Generalized） | SKILL.md §v3 核心 + feishu-delivery.md 三层结构 |
| F8 | Engineering Knowledge 独立提炼角色 | 新增 agents/engineering-knowledge-miner.md |
| F9 | knowledge_layer 字段区分两层 | contracts/knowledge-schema.md |
| F10 | 宽底座 + 窄尖顶数量纪律 | 100+ Facts → 40~60 Eng → 15~25 Patterns → 7~12 KO |
| F11 | generalized 必须回溯 engineering 底座（防悬空升维） | knowledge-schema + synthesizer 禁则 |

## v2 → v3 度量框架

```
v2: Facts → 直接压缩成 KO（过度压缩，工程细节丢失）
v3: Facts → Engineering Knowledge（宽底座）→ Generalized KO（窄尖顶）

度量：
  三层配比 Facts : Engineering : Generalized
  Engineering 覆盖（核心机制/实现/决策/失败/配置/边界）
  KO 底座可回溯率
  工程层保留完整度（Weak ref / cancellation / drop 等机制是否保留）
```

## 与既有机制的兼容

- 不推翻 v2 的 Blind Reconstruction / Promotion Gate / 反例预算制 / Policy Flow
- 在 v2 之上增加"工程层独立保留"，防止过度压缩
- 五层阶梯（L1→L5）仍适用，但映射到三层架构：L1/L2 → Engineering；L3/L4/L5 → Generalized

## 待验证

- v3 对 Codex 重跑后：三层配比是否合理、工程层是否保留 Weak ref 等关键机制、KO 是否全部可回溯底座
- 是否会出现"工程层过宽无重点"的新问题（需 coverage-auditor 判断高价值机制与低价值细节的区分）
