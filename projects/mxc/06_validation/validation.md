# 06 · Validation & Evidence — MXC 六审 + 反例 + 质量指标

## 六审（全部先 Blind Reconstruction）

### 1. Truth Auditor（真值）
- **做法**: 独立重读仓库核心断言，不引用 EK 文本；核对每条关键事实的文件:行
- **结论**: 20 条 EK 的关键事实全部可回溯；0 条 CONTRADICTED
- **重点核验**: PolicyHash 排除表（policy_identity.rs 文档 vs 代码穷举投影一致）；AuditEventName 闭合枚举（audit.rs 枚举 vs 文档一致）；experimental 是参数非字段（experimental.rs + spawn_one_shot_json 签名）

### 2. Coverage Auditor（覆盖）
- **做法**: 独立重搜未读子系统（schemas/、docs/containment-policy、learning_mode_windows、process_container dispatcher、bwrap_command）
- **覆盖**: 引擎/公共基础/后端注册/学习模式/契约/校验/审计/遥测/独立执行器/测试体系/策略文档/版本化 schema
- **缺口（声明）**: ①未逐行深读 LXC/Seatbelt/IsolationSession 后端 runner 内部（由 dispatcher 语义覆盖）；②host/（plm/wxc_host_prep）未深读（组织策略路径，见 C-M01）；③.NET/Node SDK 包装层未深读（Rust 内核为事实源）

### 3. Flow Auditor（流真实性）
- **做法**: 逐条 Flow Edge 对照代码路径
- **结论**: 七类流关键 Edge 全部可回溯 symbol/file/condition；0 条假 Edge
- **重点**: Control（dispatch.rs match）、State（state_aware_operation 映射）、Data（probe_version）、Evidence（audit.rs sink 规则）、Authority（require_experimental_optin）、Memory（DestroyOnExit）、Policy（NetworkAction default deny + bwrap --die-with-parent）

### 4. Abstraction Auditor（升维判定）
- **判定**: KO-M01/M02/M03 保持 L4（解释范围扩大有据：跨后端一致属性）；KO-M04/M05 保持 L3（单项目模式，未达 L4 稳定关系）；全部标 Cross-project validation pending
- **降级决定**: 未把"策略生命周期闭环"升到 L5 方法论（单项目证据不足）；C-M06 留在候选
- **反退化检查**: 无"同子系统=聚合理由"（每个 KO 有明确 aggregation_rule）

### 5. Counterexample Hunter（反例）
- **攻击清单**（每个 L3+ KO ≥1 定向反例）:
  - KO-M01 反例: "PolicyHash 无密钥=确认预言机" → 已文档化接受 + 敏感输入全部排除（无漏洞级反例成立）
  - KO-M02 反例: "dry_run 可能绕过执行审计" → streaming 显式拒绝 dry_run（反例被代码 gate 击穿）
  - KO-M03 反例: "JSON 策略可写 experimental:true" → schema 无该字段（反例不成立——字段是函数参数）
  - KO-M04 反例: "dangling symlink 逃逸" → PathCanonical::Unknown 分类拒绝（反例被三态模型覆盖）
  - KO-M05 反例: "schema 就地修改" → PUBLISHED IMMUTABLE 自述 + 版本窗口校验（反例不成立）
- **0 反例 KO**: 无（每 KO 至少 1 个定向攻击，且有搜索证据）

### 6. Epistemic Auditor（认知状态）
- **检查**: Fact/Observation/Hypothesis/Pattern/Model/Principle 是否混淆
- **结论**: 无混淆——C-M01/C-M02/C-M06 明确标 Hypothesis/Candidate；KO 全部标 Cross-project pending；EK 层明确 L1/L2
- **发现**: 一处需警惕——README "agent 不能成为自身安全权威"是产品主张（C-M03 已标注为 Observation 而非引擎机制）

## 反例与矛盾保留（不因全绿弱化）
- **Contradictions**: 0 条（本次考古未发现代码内部自相矛盾）
- **Counterexamples 保留**: 上述 5 组攻击记录在案（含被代码 gate 击穿者——它们是验证证据，不是删除理由）
- **KO 反向追溯**: KO-M01~M05 全部可回溯 EK-M0x → 文件:行（03 层矩阵）

## 质量指标
| 指标 | 值 |
|---|---|
| EK 数（02 层） | 20（M01~M20） |
| EK 带 links 比例 | 100% |
| EK 平均出边 | ~1.6 |
| 游离 EK | 0% |
| KO 数（03 层） | 5（M01~M05） |
| KO aggregation_rule 覆盖率 | 100% |
| KO 簇规模 | 2~3 |
| Facts/Evidence 引用 | ~60 条（文件:行可回溯） |
| Validation 判定 | CONFIRMED 为主，0 CONTRADICTED |
| Candidates | 7（C-M01~M07，全部标注 Epistemic 状态） |

## 证据强度分布
- S3（Implemented）: 绝大多数 EK（源码实盘）
- S4（Test Validated）: 策略默认值测试（network_action_defaults_to_deny 等）、注册表防退化测试、契约版本测试、state_aware 路由测试（1598 行 exec_state_aware_routes_windows_sandbox_exec_to_backend）
- 未用 S5+（本环境无运行时执行观测）
