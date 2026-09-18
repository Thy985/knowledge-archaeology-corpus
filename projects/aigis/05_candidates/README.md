# 05 — Candidates（未验证 / 跨项目 / 范围不确定）

> 纪律：Candidates 保持 Hypothesis 标记；不冒充 Fact/Principle。

## C-01 — 交叉项目假说：确定性 guardrail 与 LLM judge 是互补而非替代
- **Hypothesis**：确定性检测（Aigis 路线）负责"可解释、低成本、高确定性"的已知攻击模式；LLM judge（agentevals 谱系，已考古）负责"需要理解意图"的未知攻击——两者的分界在"判定后果是否可逆"。
- 当前证据：Aigis 单项目（S3/S4）；agentevals 单项目（S4）。
- 缺失证据：同一攻击集上双路线的横向对比实验。
- 验证路径：构造统一基准（同 154 条攻击样本）跑两条路线，比 coverage/FP/latency/cost。

## C-02 — 暂定模式：per-category score cap（base×2）抑制噪声聚合
- **Hypothesis**：对单类别得分设上限可防止"同一攻击类别的多条弱规则累积成假阳性"，是 FP 控制的结构性手段。
- 当前证据：ARCHITECTURE L1 明示 cap 规则 + 0.0% FP 自述（S7，非实测）。
- 缺失证据：移除 cap 后的 A/B 实验数据（对 26 benign 样本的敏感性）。
- 验证路径：修改 cap 参数跑 benchmark，对比 FP/TP 曲线。

## C-03 — 范围不确定：pyaigis-kr（本快照）与上游 pyaigis 的能力差异
- **Tentative**：本仓库 = pyaigis 韩国合规 fork（v1.1.4），含 53 项韩国合规映射 + 韩文 PII 检测器 + 11 韩文注入模式 + jamo 归一化；检测管线主体继承上游。雷达卡引用的"pyaigis v1.1.11 + 44 模板"与本仓库实际（v1.1.4 + 12 模板）有版本差。
- 当前证据：pyproject description + README.ko.md + policy_templates/（kr_* 3 个）+ i18n（ko 支持）。
- 缺失证据：未安装 PyPI 上游 pyaigis 实测比对（本环境未装，仅源码 fork 关系可证）。
- 验证路径：`pip install pyaigis` + `pip install pyaigis-kr` 双实例 diff。

## C-04 — 潜在原则：安全工具的"能力边界声明"是可审计性的组成部分
- **Hypothesis（L3 候选）**：不掩饰 miss 类别（10 个 alignment-frontier 明确标注 "not claimed as solved"）比"全绿"声明更能支撑工具可信度——边界声明让使用者在已知缺口处补防线。
- 当前证据：README benchmark 表诚实声明（S7）+ L6/L7 roadmap 对应（S3）。
- 缺失证据：该边界声明是否实际影响采用决策（无用户研究）。
- 验证路径：跨项目对比安全工具 README 的边界声明模式。

## 未解决矛盾
- **M-01**：README 能力声明（93.5%/0.0% FP/60+ detectors）以**原版 Aigis**（killertcell428/aigis v1.1.0）为准，本 fork 实际检测面（韩文 PII/注入扩展）无独立基准数字——fork 与原版的能力差**未量化**（诚实标注为项目自述差异，未虚构数字）。
