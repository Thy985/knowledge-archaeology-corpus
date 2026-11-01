# 03 Knowledge Layer（Generalized KO）— deepseek-harness v0.2.1-alpha.2

> KO 是 EK 图上的簇，每个 KO 声明 aggregation_rule（R1 机制簇 / R2 因果链簇 / R3 不变量簇 / R4 主题簇）。簇成立五条件：内聚性/跨实例性/解释范围扩大/可命名/可回溯。L1→L5 逐层标注，Epistemic 状态严格区分。

## KO 清单（10）

### KO-01（L4 Cognitive Model，R4 主题簇）全插件、无特权内核的运行时组织
- **声明**：Agent 运行时把一切能力（模型/工具/会话/沙箱/循环）组织为可替换插件，并显式拒绝出现"特权内核"——包括拒绝出现特权化扩展路径。
- **簇内 EK**：EK-27（profile/bundle 分层）、EK-28（能力 seam 三角色）、EK-19（动态扩展受控）、EK-18（应用启动纪律）、EK-40（sdk-minimal 例外）
- **aggregation_rule**：R4 主题簇（"全插件组装"主题，EK 覆盖组装/替换/扩展/启动/例外五互补维度）；边类型 subsystem+mechanism+constraint
- **L3 对照**：插件化运行时模式（同类：Cordis/VSCode 扩展体系）
- **反例**：extensions 的 node:vm 动态定义是"运行时自修改"——但被约束为进程级、持久安装唯一通道 Plugin Manager（EK-19）→ 反例不推翻，反而确认"无特权内核"含"受控自修改"
- **Epistemic**：Validated Pattern（0.1.2 与 0.2.1 两版本一致 + 代码级证据）

### KO-02（L4 Cognitive Model，R2 因果链簇）turn/step 生命周期：决策先于提交
- **声明**：Agent 循环把"决策"与"提交"分离——接受输入（pre-step）、路由（prepareCall）、冻结（freeze）、流式、工具执行各自成阶段，取消或拒绝发生在提交前则什么都不落盘。
- **簇内 EK**：EK-06（生命周期）→ EK-07（pre-step 决策）→ EK-08（prepareCall 路由+取消不提交）→ EK-05（失败记录 missing tool results）→ EK-04（attempt 保留）
- **aggregation_rule**：R2 因果链簇（决策→路由→提交→失败处理 完整链）；边类型 causal+mechanism
- **L3 对照**：两阶段提交 / 预检-提交模式
- **Epistemic**：Validated Pattern（docs/architecture.md §Turn flow 与 agent-loop 实现一致）

### KO-03（L4 Cognitive Model，R1 机制簇）事件溯源会话：日志即真相
- **声明**：会话状态由 append-only 事件日志唯一决定；"model-visible means logged"；投影、重放、telemetry、持久化全从日志派生——系统不需要第二个真相源。
- **簇内 EK**：EK-01（append-only + invariant）、EK-11（projection seam）、EK-31（世代文件布局）、EK-02/03（版本化迁移）
- **aggregation_rule**：R1 机制簇（事件溯源机制跨 session/projection/storage 子系统出现）+ R3 不变量（model-visible means logged 汇聚）；边类型 mechanism+subsystem+constraint
- **L3 对照**：事件溯源（ES）/ CQRS 读模型
- **Epistemic**：Validated Pattern（跨版本维持）

### KO-04（L4 Cognitive Model，R3 不变量簇）安全边界是显式声明而非隐含保证
- **声明**：运行时把"隔离/安全"作为必须显式声明的边界而非默认保证——SAFETY.md 否定沙箱保证、PTC isolation 描述符声明"不承诺安全边界"、工具管道把关（policy/guard）是唯一强制点。
- **簇内 EK**：EK-16（SAFETY.md）、EK-15（sandbox 组）、EK-17（approval/credentials）、EK-14（PTC isolation 描述符）、EK-13（把关管道）
- **aggregation_rule**：R3 不变量簇（"隔离不隐含"安全不变量汇聚五条 EK）；边类型 constraint+mechanism
- **反例攻击**：sandbox-policy/sandbox-windows-acl 是否提供强隔离？——SAFETY.md 明言"even correctly enforced restrictions cannot protect resources the project is allowed to access"（声明边界而非保证）
- **Epistemic**：Validated（SAFETY.md 原文 + 代码结构一致）；跨项目含义与 10-11 雷达"评测隔离"同频（外部事件，见 C-02）

### KO-05（L4 Cognitive Model，R4 主题簇）持续工作控制面：目标/任务/提醒/交付四服务
- **声明**：Agent 运行时把"跨 turn 持续工作"拆为四个正交服务——目标（goal，为何做）、后台任务（jobs，怎么并行）、提醒（schedule，何时回来）、交付记录（deliverables，交了什么）——各自持久、各自挂在既有事件/seam 上，无新特权面。
- **簇内 EK**：EK-20（goal）、EK-21（jobs）、EK-22（schedule）、EK-23（deliverables）、EK-24（spill）
- **aggregation_rule**：R4 主题簇（"持续工作"主题五互补维度）；边类型 mechanism+subsystem
- **L3 对照**：后台任务/定时器/目标跟踪的运行时服务化
- **Epistemic**：Validated Pattern（五个包 README + 实现一致）

### KO-06（L3 Pattern，R1 机制簇）单例 provider 注册模式
- **声明**：设备/后端类能力（computer-use、browser-use）用"一次一个 provider"注册约束，避免多驱动竞争；provider 自带工具与操作。
- **簇内 EK**：EK-25（computer-use/browser-use 单 provider）
- **aggregation_rule**：R1 机制簇（同机制跨 computer-use/browser-use 两实例）+ 跨实例性（两独立包同模式）
- **Epistemic**：Validated Pattern（两包 README 一致）

### KO-07（L4 Cognitive Model，R3 不变量簇）应用启动纪律：单一权威入口
- **声明**：运行时把所有 Node 应用启动收敛到命名 profile 单一入口，任何绕过路径被静态检查拒绝——启动权威不被工具/脚本/进程内挂载分流。
- **簇内 EK**：EK-18（verify-application-entrypoints）、EK-19（动态扩展受控）、EK-27（profile 组装）
- **aggregation_rule**：R3 不变量簇（"启动必须经 dsh profiles"不变量）；边类型 constraint+mechanism
- **反例**：vendored CLI/test-only 可执行文件被显式分类（不是绕过而是被清单化）——EK-18 原文
- **Epistemic**：Validated（脚本存在 + docs 声明）

### KO-08（L4 Cognitive Model，R4 主题簇）版本化兼容治理：数据契约随发布升级
- **声明**：持久化数据格式（Session format）把"版本号"与"发布"绑定——alpha/rc 发布即承担 released 义务；迁移必须 adjacent、世代不可变、前代无降级承诺；唯一 writer 权威在代码常量。
- **簇内 EK**：EK-02（SESSION_FORMAT_VERSION=4）、EK-03（迁移纪律）、EK-31（世代布局）、EK-01（append-only）
- **aggregation_rule**：R4 主题簇（"版本化兼容"四互补维度）；边类型 constraint+mechanism+causal
- **Epistemic**：Validated Pattern（session-format-status.md + types.ts:89 一致）

### KO-09（L3 Pattern，R2 因果链簇）守卫闭环：卡死/挂死的显式反制
- **声明**：循环健康由显式守卫保证——重复工具调用触发换策略提醒、声明超时的工具调用限时返回明确错误；守卫是 advisory/cooperative 而非侵入性强制。
- **簇内 EK**：EK-12（guard 家族）→ EK-38（动机：卡死/挂死）→ EK-05（失败记录）
- **aggregation_rule**：R2 因果链簇（反模式→守卫→模型反馈）
- **Epistemic**：Validated（guard 源码 + 实现一致）

### KO-10（L4 Cognitive Model，R4 主题簇）模型不可见面与模型可见面分层
- **声明**：运行时把"模型可见"（prompt/tools/goals 输入）与"模型不可见"（workspace 分组、deliverables 记录、spill 存储、observer-only 日志）显式分层——UI/宿主消费模型不需要的东西，不进入请求上下文。
- **簇内 EK**：EK-26（workspace 模型不可见）、EK-23（deliverables 仅客户端读）、EK-24（spill 预览+定位器）、EK-01（model-visible means logged）
- **aggregation_rule**：R4 主题簇（"可见性分层"四互补维度）；边类型 subsystem+mechanism
- **反例**：spill 的 locator 会回到模型（作为工具结果 preview）——但那是受控的 bounded preview，非全文
- **Epistemic**：Hypothesis→Pattern 边界：可见性分层由多包 README 实证，但"全局设计意图"来自文档综合 → 标 Validated Pattern（文档+实现一致）

## 度量

- KO 数：10 ｜ aggregation_rule 覆盖率：10/10（100%）｜ 平均簇规模：5.0（10 KO / 覆盖 EK 约 50 条次，含共享）
- 规则分布：R1×2（KO-03/06）、R2×2（KO-02/09）、R3×2（KO-04/07）、R4×4（KO-01/05/08/10）
- L4 KO：8 ｜ L3 KO：2 ｜ L5：0（无跨项目充分证据，不升 Methodology）

## 纵向链示例（L1→L4 全链成立）

```
EK-16 SAFETY.md 显式声明沙箱不保证隔离（L1/L2 engineering）
  ↓
KO-04 安全边界是显式声明而非隐含保证（L4，R3 不变量簇）
```
```
EK-20 goal continuation 权限进程级 + EK-21 jobs owner fence（L2）
  ↓
KO-05 持续工作控制面：持久状态与进程级权限分离（L4，R4 主题簇）
```
