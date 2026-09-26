# 01 · Project Layer — Aigis 项目地图

> L0 事实层：是什么、怎么运行、模块边界。全部可追溯到仓库 HEAD 5874cbb5。

## 1.1 项目定位

Aigis 是 **企业引入 Claude Code 及其他自主 agent 的信任层（trust layer）**（ARCHITECTURE.md Overview）。监控输入/输出/MCP 工具定义，跨 25+ 威胁类别检测与阻断，并生成安全审查要求的 settings 与审批文档。

定位的精确表述（llms.txt，机器可读摘要）："an independent, Apache-2.0, zero-dependency Python trust layer for adopting Claude Code and other autonomous AI agents inside a company"——每个工具调用上加确定性护栏、防篡改审计日志、从实时配置生成 EN/JA 审批包。

## 1.2 架构总览

```
AI Agents（Claude Code / OpenAI-Anthropic / LangChain-LangGraph / Custom）
        │
        ▼
Aigis Security Layer
  ├─ Adapter Layer    Claude Code hooks │ FastAPI middleware │ LangChain CB │ Proxy
  ├─ Detection & Enforcement Pipeline（4 层）
  │    L1 regex（165+ patterns，25+ 类，EN/JA/KO/ZH + NFKC/零宽/空格压缩/Confusable/Emoji 归一化）
  │    L2 语义相似（56 短语，difflib + n-gram）
  │    L3 主动解码（Base64/Hex/ROT13/URL/Unicode → decode → rescan）
  │    L4 CaMeL 能力访问控制（控制流/数据流分离 + taint + capability tokens + policy enforcement）
  └─ Output Layer     Activity Stream（3 层日志）│ Remediation Hints（OWASP/CWE/MITRE）│ Compliance Report │ Benchmark/Badge
```

（ARCHITECTURE.md Overview + Module Structure）

## 1.3 核心模块边界

| 子系统 | 模块 | 职责 |
|--------|------|------|
| 检测引擎 | scanner.py + filters/patterns.py + decoders.py + similarity.py | 四层检测管线的 L1-L3 |
| 能力访问控制 | capabilities/{enforcer,taint,store,tokens}.py | CaMeL 分离（L4）+ 授权决策 |
| 策略引擎 | policy.py + policies/manager.py | 声明式 allow/deny/review 规则 + 条件扩展 |
| 审计 | audit/{signed_log,chain,verify}.py | HMAC-SHA256 签名日志 + 哈希链验证 |
| 日志与流 | activity.py + forwarders/ | 3 层日志 + 异步转发（Redactor 协议） |
| MCP 安全 | mcp_scanner.py | 6 攻击面扫描 + rug pull 快照比对 + trust score |
| 记忆安全 | memory/{scanner,integrity,imitation_detector}.py | 记忆投毒扫描 + TTL/hash 完整性 + 模仿检测 |
| 多 agent | multi_agent/message_scanner.py | 跨 agent 消息模式检测 |
| 供应链 | supply_chain/{hash_pin,sbom,verify}.py | 工具哈希固定 + SBOM |
| 适配器 | adapters/claude_code.py | hooks 生成 + fail-closed 拦截脚本 |
| 设置派生 | settings_export.py | policy → Claude Code 权限（deny/ask/allow） |
| 审批包 | trust_pack.py + compliance.py | 合规映射 + 证据收集 + EN/JA 文档生成 |
| CLI | cli.py | 约 25 子命令 |
| 服务器 | server.py | 中间件/代理 |
| 自测 | redteam.py + adversarial_loop.py + benchmark.py | 红队 + 攻击-防御-改进循环 + 内置对抗套件 |
| 运维 | weekly_report.py / report.py / badge.py / auto_fix.py | 周报/报告/badge/自动修复 |
| 自进化 | auto-improvement/（仓库顶层目录） | 6 小时循环 + 论文评审循环 |

## 1.4 关键数据结构

见 snapshot_artifact.md §核心数据结构（CheckResult / ActivityEvent / SignedLogEntry / TaintedValue / Capability / MCPToolSnapshot / ControlMapping）。

## 1.5 生命周期与版本史

- 2026-04-11：v2.0.0 上传 PyPI（后被判定"无可用 2.0.0"，号被烧）
- 0.0.x → 1.x 发展期；2026-08-24：**v2.0.1** 发布——战略转向发布（CHANGELOG：1781 tests passed）
- 2026-08-13 ROADMAP 修订：放弃 1000 stars → 日本企业安全审查工具
- 2026-09-02：最近 commit（HEAD）；GitHub 描述改为"get Claude Code and other AI agents approved for use at work"
- 当前：54★、单人维护（@killertcell428）、auto-improvement 循环每 6 小时自动改进仓库

## 1.6 配置与状态

| 配置面 | 位置 | 内容 |
|--------|------|------|
| 策略 | aigis-policy.yaml | 声明式规则（load_policy 默认路径） |
| 项目日志 | .aigis/logs/ | jsonl，7 天 gzip，60 天删除 |
| MCP 快照 | .aigis/mcp_snapshots/ | 工具定义快照（rug pull 基准） |
| 审计 | .aigis/audit_key + signed_audit.jsonl | 签名密钥 + 签名日志 |
| 学习规则 | .aigis/learned_patterns.json | auto-fix 学习的 pattern |
| 全局日志 | ~/.aigis/global/ | 跨项目聚合 |
| 警报 | ~/.aigis/alerts/ | block/review 永久保留 |
| 企业模式 | enterprise_mode=true | incidents/SLA/review replay + PostgreSQL |

## 1.7 治理机制

- 开源治理：GOVERNANCE.md（用户/贡献者/维护者分级）、DCO、CODE_OF_CONDUCT、SECURITY.md、OpenSSF Scorecard、CodeQL
- 发布治理：release.yml 拒绝 orphan tag（tag 必须 reachable from origin/master）→ CLAUDE.md 硬约束"先 merge 到 master 再打 tag"；release_preflight.sh tag 前查 PyPI（防烧号）
- 自进化治理：auto-improvement/ 是"远程维护 agent 的台账"（非人直接编辑），论文候选只进 pending/，由人 PR 升级到 aigis/
- 审计治理：`aigis audit verify` 提供完整性证明（HMAC + 哈希链）

## 1.8 外部依赖

运行时零依赖（标准库 only）；集成对象为 Claude Code/Anthropic/OpenAI/LangChain-LangGraph/MCP/Docker；可选 PostgreSQL（Enterprise）、Slack/Webhook、SIEM 转发。
