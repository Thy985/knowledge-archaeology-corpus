# Candidates — Strands Box（C-B01~B07）

> 不能确认的内容留在候选。Hypothesis 严格区分，不冒充知识。

## A 类：外部宣称 / 未在引擎验证

### C-B01 · 组织级多 box 的凭据隔离（Hypothesis）
- **claim**: "A process that must not use a credential belongs in a separate box"（credentials.md）
- **证据**: 文档主张；引擎侧 box 粒度边界明确（gateway 无法分辨请求来源进程）
- **缺口**: 未见跨 box 编排/路由的原生机制（多 box 协同是 operator 侧编排）
- **验证路径**: host/ 编排层或后续轮次；标记 NEEDS_HUMAN_REVIEW

### C-B02 · Linux 后端与 macOS 的保证等价性（Hypothesis）
- **claim**: "Portable in definition, honest in guarantees"（tenet 5）；preview 声明 macOS only
- **证据**: crates/containment/src/backend/linux/namespace/* 代码存在（netns/reaper/syscall/probe）；README "currently supports local execution on macOS with Apple silicon"
- **缺口**: Linux 后端未达 GA 保证；两平台保证差异（如 Linux namespace vs Seatbelt 的网络强制面）未逐一对照
- **验证路径**: 周五 MXC×Box 双对照时跑 Linux 后端（MXC Linux 侧可对照）

## B 类：跨项目假说 / 暂定模式

### C-B03 · 沙箱范式对照：OS 层策略强制 vs 容器/微VM 强隔离（Hypothesis，跨项目）
- **claim**: OS 层遏制（Box）能执行"上下文规则"（read-only 可读生产日志但不可改），容器/microVM 只能限触达
- **证据**: 候选卡 + Box 文档（"sandbox can limit what you can touch, not what you can do with what you touch"）
- **对照**: MXC 多后端谱系（Bubblewrap/AppContainer/Seatbelt/LXC/WSLC/MicroVM）——同域两大厂方案
- **缺口**: 需 MXC×Box 实测对照（Friday 验证计划①）

### C-B04 · 策略语言范式对照：JSON 声明 vs Cedar/Dogwood 语法（Hypothesis，跨项目）
- **claim**: JSON 策略（MXC containment-policy）利于机器生成/校验；Cedar 派生语法（Box/dogwood）利于时间性/复合条件表达
- **证据**: 两项目 policy 形态；dogwood 语言本体考古（09-08）
- **缺口**: 工程可用性/表达能力对比需实证

### C-B05 · 遥测范式对照：OTLP 决策记录 vs 单行 JSON audit（Hypothesis，跨项目）
- **claim**: Box 决策记录=OTLP JSON（records.jsonl，enforcement point 提交 canonical）vs MXC 审计=单行 JSON 诊断（永不达 SDK 输出）
- **证据**: telemetry.md + mxc 考古（audit.rs）
- **缺口**: 两种遥测哲学的适用边界未实证（决策可观测性 vs 审计最小披露）

### C-B06 · 无状态解释器作为 agent 工具执行的通用安全模式（暂定 Pattern）
- **claim**: "no interpreter state crosses a call boundary" + 全部 I/O 经决策 = agent 工具执行的通用降险模式
- **证据**: Box 实现（decisions.md:262）+ 架构一致性
- **缺口**: 跨项目验证（dsh/AGT/TeamMind 工具执行对照）；单一项目证据不足以 L3→L4

### C-B07 · 403+rule id 反馈环：拒绝可观测性让 agent 自纠（暂定模式）
- **claim**: 拒绝携带 rule 标识与描述，agent 可自适应调整（等待而非重试）
- **证据**: 候选卡+决策记录机制（无 approver 判定）
- **缺口**: agent 自适应行为未见基准数据（behavioral benchmark pending）
