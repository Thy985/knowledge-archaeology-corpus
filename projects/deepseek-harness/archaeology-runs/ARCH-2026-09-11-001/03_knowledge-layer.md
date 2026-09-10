# 03 — Knowledge Layer（Generalized KO，v0.1.5 REFRESH）

> 规则：每个 KO 声明 aggregation_rule（R1 机制簇 / R2 因果链簇 / R3 不变量簇 / R4 主题簇）+ 簇内 EK + 解释范围扩大论证。禁止"同子系统=聚合理由"。

## 3.0 基线 KO 存续性核对

| KO | 主张 | 状态 | 本轮补强 |
|---|---|---|---|
| KO-01 | 日志驱动可恢复回合循环（R2） | 成立 | V3 envelope（EK-R07）强化"重建无分歧" |
| KO-02 | Capability Seam 三件套（R1） | 成立 | workspace-files 是第 N 个实例（EK-R04） |
| KO-03 | 权限 Fail-Closed 不变量族（R3） | 成立 | PTC collapse 在策略前拒绝（EK-R03）、读权限继承 fs 后端（EK-R05）同族 |
| KO-04 | 四层上下文记忆治理（R4） | 成立 | 未发现变化 |
| KO-05 | 日志即真相：可重放与审计（R3） | 成立 | V3 拒绝矛盾元数据推断（EK-R07） |
| KO-06 | 策略治理闭环（R2） | 成立 | 09-05→09-09 supersede 是闭环实例（EK-R05） |
| KO-07 | 结构化失败族（R1） | 成立 | UNKNOWN_TOOL/TypeError/ABORTED_BEFORE_DISPATCH 细分（EK-R03） |
| KO-08 | 可插拔系统治理护栏（R4） | **深化** | Agent Notes 体系=决策护栏（EK-R01）；SAFETY.md 责任声明（EK-R10） |

## 3.1 新增 KO

### KO-09 — 呈现/执行双面强制：wire 收缩必须由 executor 执行（R3 不变量簇）
**aggregation_rule**: R3（不变量簇）——约束边汇聚到同一安全不变量。
**簇内 EK**: EK-R03（collapse 经 resolveExecution）、EK-07（受控管线）、EK-10（approval fail-closed）、EK-09（guard 单调拒绝）。
**不变量**：*对模型隐藏的东西，执行面也必须拒绝——schema 省略不是强制，绕过路径必须在执行边界被确定性截断，且先于任何策略/审批/守卫。*
**解释范围扩大**：不只 dsh——任何"呈现面收缩"的 Agent 系统（工具过滤、mode、白名单）都要求执行面同构；Direct caller bypass 是通用反模式。
**反例检查**：collapse 前版本（schema 省略但 executor 放行）即该不变量的反例与修复动机；nested=true（SDK 绑定）是"合法例外"——异常必须可归因（parent token 仅 SDK 设置）。
**L4**: 呈现声明与执行授权必须共享同一个折叠谓词，否则最小面承诺被直呼路径击穿。

### KO-10 — 权限边界按"操作语义"分层而非"单一边界"（R4 主题簇）
**aggregation_rule**: R4（主题簇）——覆盖同一主题互补维度。
**簇内 EK**: EK-R05（read 继承 fs 授权 / list·changes workspace-scoped）、EK-R04（服务职责分离）、EK-R08（iframe 权衡）、EK-12（sandbox 三态）、EK-13（credential ref）、EK-14（presets 两旋钮）。
**模型**：对**具名资源读取**，边界=底层后端授权（workspace 只是路径基准）；对**导航/观察**（list/changes），边界=workspace；对**执行**，边界=sandbox+approval+guard；**预览/渲染**，边界=opaque origin + 显式网络取舍。每种操作语义有自己的边界，且边界选择**写成 ADR 并记录 Alternatives**。
**解释范围扩大**：dsh 三处独立验证（09-05→09-09 supersede、readRelated `..`、iframe CSP）证明"权限=单一路径"不成立；其他文件预览/沙箱系统可迁移该分界法。
**反例**：严格 containment（09-05 初版）被证明过窄（阻断显式可读文件、破坏 HTML 外链）；CSP 禁网被证明过严（拒绝有意保留行为）。

## 3.2 五层定位（本轮新增）

| KO | L3 Pattern | L4 Cognitive Model | L5 |
|---|---|---|---|
| KO-09 | 呈现/执行双面强制模式（同类：白名单+执行校验、eBPF 安全策略） | 可见性收缩必须带执行面强一致 | 任何面缩小改动必须补 executor 级拒绝测试（S4 起） |
| KO-10 | 按操作语义分层的权限模型（同类：POSIX read vs directory traversal 语义、OAuth scope 分层） | 权限边界是操作语义的函数，不是单一字符串路径 | 权限设计先列操作语义清单，再逐类定边界，并以 ADR 记录权衡 |

**升维纪律**：KO-09/KO-10 均为跨项目 Candidate（Cross-project validation pending），未宣称 Principle/Law；L4 只写"稳定关系"，L5 只写可操作准则。

## 3.3 三层配比

- EK 45（32+13）/ KO 10（8+2）/ Candidates 14（9+5）——宽底座保持，尖顶窄化（2 个新 KO 均有 cross-project pending 标注）。
- 所有新增 KO 可回溯：KO-09→EK-R03/R07；KO-10→EK-R04/R05/R08（evidence 链完整）。
