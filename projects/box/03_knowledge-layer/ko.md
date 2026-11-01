# Knowledge Layer — Strands Box（KO-B01~B05）

> 每条 KO 声明 aggregation_rule（R1-R4），簇内 EK + 边类型；L3/L4 严格区分 Epistemic 状态。

## KO-B01 · 授权面唯一化 + 地板分离（L4 Cognitive Model，R3 不变量簇）
- **聚合**：EK-B01（CapabilityOutcome 无 Allow/Deny）+ EK-B03（deny-only floors 只能减）+ EK-B06（单门面）+ EK-B20（加载期拒绝）
- **边**：constraint 边汇聚到同一安全不变量："**只有一个能说 yes 的面；一切 else 只能 subtract**"
- **认知模型（一句话）**：安全系统里，授权必须收敛到唯一可判定点，其余机制降级为只能收窄的 floor，且类型系统承担"机制不能授权"的约束
- **反例压力**：Box 自身（interpreter 无 credential floor——暴露面诚实保留）；MXC 对照（解析器层硬编码 experimental=false 是不同侧的双保险）
- **Epistemic**: Validated Pattern（Box 内多子系统一致：network/interpreter/mcp 三路径同构）；Cross-project validation pending（MXC/dsh 对照）

## KO-B02 · 凭据 = 注入点的配置，不是授权（L4，R3 不变量簇）
- **聚合**：EK-B08（占位符换真）+ EK-B09（终止连接）+ EK-B11（box 是凭据边界）+ EK-B07（Zeroizing）
- **边**：constraint 边汇聚到"凭据不出 agent 进程"
- **认知模型（一句话）**：凭据绑定描述"哪个秘密能搭上哪条已授权的出站请求"，它自身不产生任何权限——注入点与决策点分离
- **反例压力**：inject=always 无占位符仍 advisory（attach.rs:381）；response 回显擦除
- **Epistemic**: Validated Pattern；与引擎 eval 层"凭据注入白名单"设计同构（candidate C-B5）

## KO-B03 · 跨调用因果的策略表达（L3 Pattern，R1 机制簇）
- **聚合**：EK-B04（单引擎单历史）+ EK-B05（temporal 规则）+ EK-B14（无状态解释器）
- **边**：mechanism 边（共享"单一历史"实现机制）跨 3 独立子系统（policy engine / enforcement points / broker）
- **模式**：把"agent 已做过的事"变成策略输入（频次/冷却/顺序），用共享事件历史跨通道关联（fs:read → net deny）；加载期拒绝不可触发规则防止策略静默失效
- **对照**：RAMPART 终局证据分离（服务端）；dogwood 语言本体
- **Epistemic**: Validated Pattern（Box 实现+文档示例）；L4 升维待跨项目验证（candidate C-B6）

## KO-B04 · 执行面与决策面分离（L4 Cognitive Model，R2 因果链簇）
- **聚合**：EK-B13（解释器在可信进程）+ EK-B15（trampoline 单向遏制）+ EK-B16（先解析后判定）+ EK-B18（叶沙箱）+ EK-B19（grants 固定披露）
- **边**：causal 链（agent→alias→broker→enforcement point→policy→effect），决策在链上单一位置
- **认知模型（一句话）**：被遏制对象永远不能自己判定自己的动作；判定发生在它够不到的进程里，且对解析后的对象执行（防 TOCTOU 式分流）
- **对照**：MXC 引擎 dispatch 单点；dsh 受控工具面
- **Epistemic**: Validated Pattern（Box 全链路一致）；L4 稳定

## KO-B05 · 单向遏制 + 一次性定形（L3 Pattern，R1 机制簇）
- **聚合**：EK-B15（trampoline）+ EK-B19（grants 固定）+ EK-B10（内核两端口）
- **边**：mechanism 边（"进程启动前一次性固定边界"）跨 macOS/Linux 两后端
- **模式**：边界配置在启动前一次性构建且终身不变；若平台沙箱单向不可撤销，用专用 trampoline 进程自遏制后 exec；启动披露 grants（因为 OS 路径无逐请求决策覆盖）
- **Epistemic**: Validated Pattern（Box 双后端 + 披露协议）；跨 OS 保证差异诚实声明（candidate C-B2）

## KO 图指标
- 5 KO（B01~B05）；聚合规则 R1×2 / R2×1 / R3×2；簇规模 3~5 EK；覆盖率 100%（每个 KO 有 aggregation_rule）
- 无"同子系统=聚合理由"的假聚合；每 KO 可回溯 EK 簇与 Evidence
