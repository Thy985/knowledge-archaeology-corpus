# 05 · Candidates · ReviewBench

> 不能确认的内容留在候选，不冒充已验证知识。每条标注：假设内容 / 当前证据 / 缺失证据 / 验证路径 / 认知状态。

## C-01 · classifier prompt 重校准周期与分数漂移的关系（Hypothesis）
- **假设**：CLASSIFIER_SYSTEM_PROMPT（v3.0.0）是"评审品味"的编码（prompts.ts L6-10 自述"most likely to need periodic recalibration"）；若底层 judge 模型更新或新品类 issue 出现，未重校准的 prompt 会系统性漂移 TP/FP 判定，导致同一 agent 跨时间分数不可比。
- **当前证据**：prompt 有独立版本号且注释明确预期重校准；METHODOLOGY §8 承认 classifier 依赖；golden 与 unmatched 分类用同一固定 prompt。
- **缺失证据**：无"prompt 变更前后对同一 agent 分数差异"的公开对照数据；无 drift 监控机制在仓库内。
- **验证路径**：跑同一 findings 集于 prompt v2.0.0 vs v3.0.0，量化分数差；或追踪 judge 模型版本变更后的复标一致率。
- **认知状态**: Hypothesis（未验证）

## C-02 · 只读工具仍可能存在信息泄漏面（Hypothesis）
- **假设**：classifier 工具集（read/grep/find/ls，classify.ts L52-53）"只读不写"防副作用，但 read 路径若含符号链接可指向 repo 外文件（mirror 冻结 checkout 理论上无外部链接，但 corpus 仓库若含 submodule/link 文件则未验证）；grep 的大目录遍历也可能被超时预算意外截断。
- **当前证据**：git.ts L30-31 声明 checkout "marked safe through GIT_CONFIG_*"；无符号链接清理逻辑（clean -ffdx 会删 untracked 但 symlink 文件本身是 tracked 内容则不删）。
- **缺失证据**：无针对 symlink/逃逸的测试或审计记录。
- **验证路径**：在测试 PR 的 checkout 中注入 symlink 指向外部文件，观察 classifier 能否读取。
- **认知状态**: Hypothesis（未验证，低影响面）

## C-03 · digest-pinned 镜像 ≠ 模型权重可复现（Hypothesis，对长期榜单可复现性有影响）
- **假设**：manifest image digest 保证"同一字节镜像"，但 agent 镜像内部调用的 LLM API（model id 指向的服务端模型）可被 provider 静默更新——同一 digest 镜像在不同日期跑，背后模型可能不同，跨日榜单可比性存疑。
- **当前证据**：schema image 字段强制 digest（"bytes cannot change under us"）；model_api_url 是运行期端点；无模型版本锁定机制（judge 侧同：provider/model id 无 API 版本 pin）。
- **缺失证据**：仓库无"模型行为版本"概念；无跨日一致性测量。
- **验证路径**：对同一 PR 同 digest 镜像跨月跑两次，比对 findings 差异。
- **认知状态**: Hypothesis（ReviewBench 文档未声称处理此点；Methodology §9 Reproducibility 需核对范围）

## C-04 · judge 模型供应链攻陷不在 KO-05 三不变量覆盖内（Hypothesis）
- **假设**：KO-05（可复现×独立×防操纵）不覆盖"judge 模型权重被污染/后门"——若 judge 本身被攻陷，全部指标失真且三个不变量检测不到。
- **当前证据**：KO-05 的防操纵面只覆盖被测者→judge 的注入（egress/注入防护），不覆盖 judge 自身可信性；仓库无 judge 模型输出一致性审计。
- **缺失证据**：无 judge 行为监控/回归检测。
- **验证路径**：对照另一 judge 模型复跑同 findings，比对一致性；或对 judge 注入已知答案探针。
- **认知状态**: Hypothesis（未验证）

## C-05 · 跨项目假说：golden 由同族 LLM 生产者主导 → 新模型"风格相似"可得高分（Hypothesis）
- **假设**：golden 的 TP 中 claude-sonnet-4.6（46.0%）+ ccr:gpt-5.5（36.5%）占 82.5%（METHODOLOGY §5.5 full set 表）；若新 agent 的 finding 表述风格与 golden 生产者同族（同为 frontier LLM），matcher 可能系统性更易判为同源 → 对"表达风格"而非"实际发现能力"的隐性加分。这是跨项目假说（影响所有 LLM-judged 基准）。
- **当前证据**：§5.5 构成表 + §8 producer bias 承认"与 producer 共享模型的 reviewer 可能受益"；无风格影响对照实验。
- **缺失证据**：无"同 finding 不同措辞 vs matcher 匹配率"的消融。
- **验证路径**：取 100 条 golden TP，改写措辞（保持语义），测 matcher 匹配率变化。
- **认知状态**: Hypothesis（跨项目候选，ReviewBench 自身 §8 已提示方向）

## C-06 · 离线 judge 与在线人类接受度的收敛未在仓库内闭环（Hypothesis）
- **假设**：offline TP/FP（Claude Sonnet 5 判定）与在线人类对 review comment 的反应（采纳/解决率）是否收敛是经验问题（METHODOLOGY §8 "Offline / online convergence" 自述"empirical question"）；仓库内无在线↔离线对照工具（Blog 报告了离线↔在线 A/B 方向一致，但该对照非仓库可复现流程）。
- **当前证据**：Methodology §8 承认 + Blog 报告离线/在线方向一致（precision/recall 增益同号）；仓库无对照 pipeline。
- **缺失证据**：可复现的在线信号收集/比对流程。
- **验证路径**：设计 comment 采纳率采集器，与 offline 标签做 matched-sample 对照。
- **认知状态**: Hypothesis（官方自述未决，本项目确认其未闭环）

---
**候选纪律**：以上 6 条均未进入 03 Knowledge Layer；无一条被写成已验证结论。C-05/C-06 为跨项目假说，保留在 Candidates 直到出现第二个项目的验证证据。
