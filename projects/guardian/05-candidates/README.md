# 05 — Candidates（未验证 / 跨项目 / 范围不确定）

> 纪律：Candidates 保持 Hypothesis 标记；不冒充 Fact/Principle。

## C-01 — 假说：Task token 强制（Phase 4）前的 fail-open 窗口
- **Hypothesis**：无 task_token 的请求跳过 ACL（"unconstrained for backward compatibility"）——在 Phase 4 全量强制前，未带 token 的 Agent 实际不受最小权限约束；若该窗口长期存在，ACL 价值被稀释。
- 当前证据：app.py:611-617（"No token — skip check (backward compat)"）——S3。
- 缺失证据：实际部署中 token 携带率；Phase 4 时间表。
- 验证路径：观测 /check 日志中无 token 请求占比。

## C-02 — 暂定模式：novel sequence 一律 sandbox 的可用性代价
- **Hypothesis**：Sequence contract 前缀匹配对"合法新行为"（未注册 playbook 的正常操作）同样 sandbox——防御收益 vs 合法创新误伤需权衡；无注册序列 agent 完全不受限，可能被绕过（agent 不注册即可规避 check 4）。
- 当前证据：app.py:896-915（"No sequences registered — agent is unconstrained"）+ novel → sandbox——S3。
- 缺失证据：真实 Agent 工作负载中 novel sequence 占比；绕过注册的实际利用。
- 验证路径：在真实 agent trace 上回放 check 4，统计 sandbox 误伤率。

## C-03 — 跨项目：Guardian sidecar + Aigis middleware 的组合形态
- **Hypothesis（Cross-project）**：两种部署形态互补——Guardian（sidecar）提供框架无关的**网关级**强制（任何框架的调用都过网关注入检查），Aigis（middleware）提供**进程内**深度检测（文本/记忆/审计细节）；组合可形成"网关拦截 + 进程内检测"双层防线。
- 当前证据：Guardian EK-13（sidecar）+ Aigis-middleware（嵌入）——S3 双项目。
- 缺失证据：组合部署的实际集成测试（无项目同时集成两者）。
- 验证路径：同一 Agent 框架上分别/同时接入两者，对比延迟与检出。

## C-04 — 范围不确定：版本号不一致（pyproject 0.1.1 vs README "4.0.0"）
- **Tentative**：pyproject version=0.1.1（Alpha）与 README quickstart 的 `"version": "4.0.0"` 不一致——可能 README 超前（新版本未发）、或服务版本独立于包版本、或文档漂移。
- 当前证据：pyproject.toml:7（version = "0.1.1"）+ README.md quickstart（"version": "4.0.0"）——S3。
- 缺失证据：CHANGELOG.md 版本史（浅克隆 depth=1 未取历史 tag）；PyPI 发布状态。
- 验证路径：git ls-remote --tags + PyPI 查询 legionforge-guardian 发布版本。

## 未解决矛盾
- **M-01**：Guardian 宣称 "No LLM. No heuristics."（README），但 check 3 的 9 个 regex 族本质上是启发式规则（模式匹配）——"无启发式"指**无概率/ML 启发式**还是**无任何模式启发**存在语义歧义；从实现看是"确定性规则"而非"统计启发式"，建议表述为 "No ML. No probabilistic heuristics."。
