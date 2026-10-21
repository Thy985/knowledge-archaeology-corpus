# 05 Candidates — agent-aegis（ARCH-2026-10-22-001）

> 不能确认的内容留在候选。Hypothesis ≠ Fact；Cross-project Candidate 不写成 Principle。

## C-G01 · AGEF/AGP 生态采纳（Hypothesis，A 类）
- **现状**: 仓库内 Draft 标准 + conformance 自测（specs/ + tests/conformance/）——自我一致
- **缺失证据**: 第三方实现者/采用者/工具链支持；specs 文件仅本仓库引用
- **验证路径**: 搜索第三方 AGEF/AGP 引用；检查 docs/comparisons 与 security docs 是否有生态证据；README 徽章仅证明 OpenSSF 参与
- **判定**: 标准真实性三层的"生态采纳"层——缺证据不升 Principle（KO-G05 已标注）

## C-G02 · autopolicy LLM 生成策略的可靠性（Hypothesis，A 类）
- **现状**: OpenAI/Anthropic generator（description→dict）+ _validate_policy_dict 校验（core/autopolicy_llm.py）
- **缺失证据**: LLM 生成策略的评测（生成质量/失败率/安全边界）；_validate_policy_dict 的完备性（能否漏放危险规则）
- **验证路径**: 跑生成器样例对照人工策略；检查校验函数覆盖字段
- **判定**: "LLM 生成治理策略"是可观察模式但本项目证据强度 S2-S3（设计+实现，无评测）——Cross-project pending

## C-G03 · 无 assessor 时的降级语义（Hypothesis）
- **现状**: `if self._assess and not claim.is_assessed: assess`（claim_policy.py:155-157）——_assess=False 时声明字段未经独立评估
- **假说**: 该路径用于低成本部署（trusted agent 场景）；但 justification_gap 为默认值 → 审批判据弱化
- **验证路径**: 搜索 _assess=False 的配置入口/文档；grep "justification_gap" 在 guardrails/action 路径的默认语义
- **判定**: 降级路径存在（代码事实），"是否设计意图"为 Hypothesis

## C-G04 · MCP shadow/escalation 检测在真实多 server 环境的有效性（Observation→Hypothesis）
- **现状**: 检测器实现明确（4 类向量 / source→sink 滑动窗口）+ 类型标注
- **缺失证据**: 基准/评测数据（误报率/漏报率）；trusted_servers 默认值行为
- **验证路径**: 检查基准测试（benchmarks/）；搜索 trusted_servers 默认
- **判定**: 机制为 Fact，有效性为 Hypothesis（无评测数据）

## C-G05 · 与 agent-governance-toolkit 的对照（Cross-project Candidate）
- **现状**: aegis = 进程内插桩 + 策略引擎 + 审批门 + 审计（一体化）；agent-governance-toolkit = 无状态策略引擎（服务端）
- **假说**: "进程内插桩治理" vs "服务端无状态策略"存在接入摩擦 vs 强制性的权衡；aegis 低摩擦但依赖 agent 进程内被 patch（bypass 面），toolkit 强制但需网络往返
- **验证路径**: 并置两 corpus 条目做对照；与 Archon 控制面（server 内 gate）三方对照
- **判定**: Cross-project pending

## C-G06 · 缓存语义与分布式一致性（Hypothesis）
- **现状**: LRU 缓存进程内（OrderedDict）；redis/postgres 为 optional extras（rate_limiter_redis 等存在）
- **假说**: 多进程部署下 policy 缓存无共享语义（每进程独立 256 LRU）——策略热更新（policy_git.py 存在）与缓存一致性关系未明
- **验证路径**: policy_git.py 实现（热更新机制）+ 缓存失效时机
- **判定**: Hypothesis（未读 policy_git 实现细节）

## C-G07 · fail-safe 的"destructive action"判定边界（Hypothesis）
- **现状**: AGP Level1 conformance 钉住 "BLOCK/DENY on destructive actions when no rules match"
- **缺失证据**: "destructive"分类器（hazard_classifier.py）的完整类别表；无规则匹配时非 destructive action 的默认（approve？）
- **验证路径**: hazard_classifier.py 类别；policy.py defaults 默认 approval
- **判定**: conformance 行为为 Fact；分类器完整面为 Hypothesis（hazard_classifier 未深读）
