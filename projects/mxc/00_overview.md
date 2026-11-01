# 00 · Overview — MXC Knowledge Archaeology（initial, ARCH-2026-10-19-001）

## 一句话定位
MXC（Microsoft eXecution Container）是微软的策略驱动跨平台沙箱代码执行系统（Rust，MIT，GA 1.0.1）：应用把不可信代码（模型输出/插件/工具）交给 SDK，SDK 按统一 JSON 策略在 OS 原生隔离后端（Bubblewrap/AppContainer/Seatbelt/LXC/WSLC/IsolationSession/MicroVM…）中执行——**"agent 不能成为自身的安全权威"**。

## 为什么考古它（Harness/治理 周 A 级缺口）
- 2026-10-07 GA 的平台级执行边界原语：策略在 agent 工作负载控制之外，由 OS 独立强制；agent/生成代码/插件/工具无法自我提权
- 与已考古 deepseek-harness（sandbox/approval seam）、agent-governance-toolkit（无状态策略引擎）形成**治理范式对照链**：dsh=应用层插件 seam、AGT=服务端策略引擎、MXC=OS 级强制原语
- 直接衔接用户项目：Tafcm（Linux Bubblewrap 后端）、TeamMind（workload+组织双层策略）、campus_order（OpenClaw 已首批集成）

## 核心认知（一句话版）
1. **策略身份 = 规范化哈希**（PolicyHash）：allow-list 投影 + 穷举 destructure + 凭据材料排除——"策略可审计且不泄露"
2. **默认拒绝是策略语言的语法**：字段省略 = 最严格；`NetworkAction::default() = Deny`；UI 全默认 deny
3. **路径规范化是文件系统策略的根基**：全别名 canonicalization（symlink/junction/8.3/\\?\）堵死别名绕过
4. **契约 = 不可变发布物**：schema 版本化 + 精确匹配，发布即冻结（"PUBLISHED IMMUTABLE FILE - DO NOT EDIT"）
5. **学习模式 = 先观察后强制的策略打磨**：DenialAnalyzer 把平台原生拒绝事件解码为跨平台 DeniedResource
6. **实验性 = 调用方 opt-in，不是 JSON 字段**：配置不能自我授权实验性访问

## 关键文件
- 01_project-layer：架构 delta + 模块地图 + 数据结构
- 02_ek_graph：EK-M01~M20（EK Graph，六类边）
- 03_ko：KO-M01~M05（aggregation_rule R1-R4）
- 04_flow_atlas：七类流（Control/State/Data/Evidence/Authority/Memory/Policy）
- 05_candidates：C-M01~M06（Hypothesis/Observation 严格标注）
- 06_validation：六审 + 反例 + 质量指标
- run_metadata.yaml：run 级元数据

## 事实来源
全部可回溯 HEAD `fda29ac63bc306b7f97f3b86b4b63d44f7f40cf2`（main）仓库实际内容（文件:行）。
