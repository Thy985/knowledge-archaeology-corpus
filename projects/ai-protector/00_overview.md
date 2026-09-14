# 00 — Overview：AI Protector（v0.2.8）

## 一句话定位

**AI agent 安全运行时**——在每次 model call 与 tool action 路径上插入的**确定性防护层**（No LLM in the loop · ~50ms · fully local · **provable**），用「先测出漏洞（Benchmark Hub）→ 再执行强制（7 层防火墙 + Agent 双门）→ 后证明守住（客观真值 grader）」闭环替代"祈祷式"防护。

## 关键数字（均为项目自述 S7，非本地实测，除标注外）

| 指标 | 值 | 模式 | 证据 |
|---|---|---|---|
| JailbreakBench（698 artifacts） | 99% | 全模式 | README + docs/BENCHMARKS.md |
| promptfoo（1103 攻击） | 65% / 91% | off / pre_llm·post_llm | BENCHMARKS.md:14-16 |
| 误报（440 benign） | 0% / 0.7% | off / harm guard | BENCHMARKS.md:171-177 |
| 延迟 p50 | 48 ms（balanced）/ ~450ms（pre_llm） | | BENCHMARKS.md:14-16,120 |
| 内部套件（358 场景 38 类） | 99.1% | CI | BENCHMARKS.md:167 |
| grader 客观准确率 | 94% → 99%（修复后） | planted canary/secret 真值，no LLM-as-judge | README + docs/red-team-oracle-calibration.md |
| 测试/覆盖率 | 1900+ tests / ~83% line | CI 声明 | README Trust 表 |

**本地实测**：668 个单元测试通过（proxy-service 纯逻辑 203 + agent-demo 465，S4）；358 场景与 DB 依赖测试因无 Postgres/Redis 不可复现（环境依赖，非产品失败）。

## 三条产品线（Find → Protect → Prove 闭环）

1. **Security Scan（Benchmark Hub）**：~5,070 场景（6 公开数据集归一化同一 taxonomy + ~110 curated）→ 指向任意 OpenAI-compatible 端点 → 种子采样可复现 → 置信度校准判定（mechanical/exact vs heuristic）。
2. **Proxy firewall**：7 检测层（Rules → Intent classifier(A2 反混淆 + ~80 regex) → LLM Guard(DeBERTa/DistilBERT) → Presidio PII → NeMo Guardrails(FastEmbed) → Jailbreak ML(DistilBERT) → Harm ML(granite-guardian 2B, strict/paranoid)）→ 加权风险分 → ALLOW/MODIFY/BLOCK 三态。LangGraph 9 节点编排，扫描器 asyncio.gather 并行。
3. **Agent gates（RBAC）**：pre-tool（权限/参数注入扫描/上下文风险/预算/人工确认）+ post-tool（PII/secret 脱敏/间接注入检测/大小限制）双门 + Agent Wizard 生成 rbac.yaml/config.yaml。

## 品类位置（跨项目对照——雷达"防火墙三选一实测"第 3 点）

| 维度 | Aigis（09-13） | Guardian（09-14） | **AI Protector（本轮）** |
|---|---|---|---|
| 嵌入形态 | middleware 内嵌 | FastAPI sidecar | proxy 服务 + agent 内嵌双门（两层） |
| 检测方法 | L1-L7 确定性规则 | 7 项确定性检查（regex 族） | **确定性规则 + 本地 ML 模型**（LLM Guard/Jailbreak ML/Harm ML）+ A2 反混淆 |
| 判定输出 | allow/deny/review | halt/sandbox/log-only | **ALLOW/MODIFY/BLOCK**（含 MODIFY=改写） |
| 决策 | 阈值 | fail-closed hot path | 加权风险分 ≥0.7 阻断 + 分级权重 |
| 审计 | HMAC-SHA256 链 | SHA-256 hash chain + canary 蜜罐 | request traces + 置信度校准报告（无 hash 链） |
| 输出侧 | exfil 检测 | 无 | **输出过滤**（PII/secret/system-leak 红线） |
| 可证明性 | 无 | 无 | **Benchmark Hub + 客观真值 grader（no LLM-as-judge）** |

## 最重要发现（Top 5）

1. **"Provable" 是品类内独有的工程承诺**：不满足于"我们有规则"，而是把 benchmark 做成产品的一等公民（Benchmark Hub + grader 用 planted canary/secret 客观真值校准 94%→99%）——与 09-15 雷达 Proof-of-Guardrail（TEE attestation）主题呼应。
2. **A2 反混淆层**：build_scan_text 把 raw+解码变体（leet/零宽/间隔/homoglyph/ROT13/base64）组合成检测视图，**原文永在第一位，LLM 永不见变体**；实验证据：variants-first 曾降检测 ~13pp。
3. **三态响应 + 分级决策**：MODIFY（PII mask/transform）是 Aigis/Guardian 之外的第三种响应——不是全有全无。
4. **NeMo 修复史**：语义 rail 从硬阻断改为风险分软贡献（硬阻断曾误伤 "search products laptop" 类良性查询）——规则层容错。
5. **双重防线架构**：proxy 层（7 层）拦截"内容攻击"，agent 层（RBAC 双门）拦截"动作攻击"——对应 README 核心命题"安全不是模型说什么，而是模型做什么"。

## 已知局限（诚实声明）

- ISS-003/004：UI 上存在**幻影功能**（ML Judge chip、Canary Tokens chip 无 backing 实现）——设计意图 ≠ 实现现实
- ISS-005/012：intent 分类器 keyword-only，改写/混淆/多语言（Turkish 6 场景 xfail）仍可绕过
- "No LLM in the loop" 语义歧义：运行时无外部 LLM API 调用，但检测依赖本地 ML 模型（DeBERTa/DistilBERT/granite-guardian）——与 Guardian "No heuristics" 张力同类
- 基准数字为项目自述（S7），本地仅复现 668 单元测试
