# 05 Candidates — microsoft/Webwright

> Hypothesis 与暂定结论留在候选，不冒充 Fact。每条含当前证据 / 缺失证据 / 验证路径。

## C-01（L3 模式假设）Skill Factory 复用增益的域限制
- **内容**：WebArena 上 held-out 55%→70% 的复用增益可能主要来自 retrieve 型模板（参数化查询 + 站点布局稳定的站点）；对交互密集/一次性长尾任务增益有限。
- **当前证据**：README 实验设定（"10 retrieve-type templates, 3 self-hosted sites, gpt-5.4"）；gate 注释自证存在 "correct-but-narrow" 风险（gate.py 文档字符串）。
- **缺失证据**：非 retrieve 类模板（form 提交、多步购物车等）的复用实验。
- **验证路径**：WebArena 全模板族按类型分组对照 reuse 增益；或对照 OpenCode 的 template 复用。

## C-02（Hypothesis）skill 库规模增长后的检索/分组伸缩性
- **内容**：retrieve（k=3, method=llm/simple）与 learn 分组（chunk=25, 模板聚类）在库规模 ≥1000 skill 时可能退化（检索噪声、分组碎片）。
- **当前证据**：retrieve.py 仅两法（llm/simple）；learn.py chunk=25 硬编码；无库规模基准。
- **缺失证据**：库规模 vs 命中率/蒸馏质量的曲线。
- **验证路径**：合成大库压力测试 + 真实多周增量观测。

## C-03（Hypothesis）live-browser 模式完成质量低于 workspace 模式
- **内容**：`require_self_reflection_success=false` 时模型自裁完成，缺少 external judge 对抗性检查，长任务上"看起来完成"的假阳性更高。
- **当前证据**：lb.yaml 显式关闭工件验证；模式差异（KO-08）。
- **缺失证据**：两模式同任务集的 A/B 完成准确率。
- **验证路径**：同一任务集分别用两种模式跑 N≥30，人工标注完成质量。

## C-04（跨项目 Candidate）"外部判定器读同一产物文件"可作评测复现锚
- **内容**：self_reflect_result.json 同时被 loop 门与 external judge 消费（base.yaml Completion Gate #4 明示），使判定可复现——这可能是一类可迁移的评测设计（judge 与 gate 共享唯一事实源）。
- **当前证据**：default.py gate 读 predicted_label；README/instance_template "The external judge reads that same self_reflect_result.json"。
- **缺失证据**：其他项目显式采用同一模式的实证（deepseek-harness 有验证门但产物格式不同）。
- **验证路径**：跨 2 项目对比"共享判定产物"模式的复现率。

## C-05（Observation）基准数字未经本考古复现
- **内容**：Online-Mind2Web 86.7% / Odysseys 60.1% / WebArena 55→70% 均来自 README 与论文声明。
- **当前证据**：README（2026-08-03 HEAD 内）；论文《Webwright: A terminal is all you need for web agents》。
- **缺失证据**：本环境无模型 key / 站点，无法实测。
- **验证路径**：有 key 环境复跑官方 benchmark 脚本（仓库未含脚本，需从论文补充）。

## C-06（Hypothesis）Skill Factory 与宿主插件（Claude Code/Codex）的协作边界
- **内容**：skills/webwright/ 共享 skill 在四宿主加载时，宿主 agent 驱动 Webwright loop 的完成门是否仍由 self_reflection 兜底，还是退化为主机自裁——插件模式下的门控强度未知。
- **当前证据**：.claude-plugin/.codex-plugin plugin.json + skills/webwright/commands；README "The host agent drives the Webwright loop natively"。
- **缺失证据**：插件运行时的实际门控行为。
- **验证路径**：在 Claude Code/Codex 内跑一次含失败 final_script 的 solve，观察 done 是否被阻塞。

## 跨项目连接注记
- C-04 与 corpus `deepseek-harness`（验证门）、`aprover`（proof 验证）存在同主题关联，待三项目对照后决定是否升维。
