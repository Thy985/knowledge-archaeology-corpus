# 00 Overview — LangGraph 考古（ARCH-2026-10-06-001）

> **核心命题**：LangGraph 让我们认识到了什么？——**有状态图是 Agent 编排的"正确抽象"：把并发、持久化、人机交互、重试全部折叠进一个版本化状态机。**

## 1. 项目一句话
LangGraph = 低层有状态多 Actor Agent 编排框架。开发者把 agent 定义为**有状态图**（节点=执行单元，边=状态转换，channel=状态聚合语义，checkpointer=线程持久化，interrupt=人机交互挂起/恢复）。v1.0 已 GA，企业用户含 Klarna / Replit / Elastic。本考古对象 commit `2d94208`（2026-10-05），版本 1.2.13，MIT。

## 2. 三层知识架构总览

### Project Layer（01）——它是什么
- Monorepo（libs/ 下 9 库）；核心 = `langgraph`（graph 构造 + pregel 执行引擎 + channels 语义原语 + managed state），底座 = `checkpoint`（持久化契约）+ `store`（长时记忆）
- 核心抽象：StateGraph（用户态图定义）→ Pregel（编译后执行引擎）→ Channel（状态聚合原语）→ Checkpoint（线程快照）→ CheckpointSaver（持久化接口）
- 生命周期：compile → invoke/stream → 超步循环 → checkpoint 落盘 → interrupt 挂起 / update_state 续跑

### Engineering Knowledge（02）——它怎么解决工程问题（48 条，EK Graph）
核心主题簇：
- **执行引擎**（EK-01~10）：超步循环、确定性写应用、版本计数乐观并发、通道语义
- **状态与生命周期**（EK-11~19）：checkpoint 结构、saver 契约、thread_id 主键、编译流水线、状态注入
- **interrupt / HITL**（EK-20~25）：挂起-恢复-重执行协议、Command 三通道
- **错误与重试**（EK-26~30）：RetryPolicy/TimeoutPolicy、异常分层、错误处理器节点
- **数据流/读写**（EK-31~35）、**序列化与存储**（EK-36~40）、**高层框架**（EK-41~44）、**测试揭示行为**（EK-45~48）

### Knowledge Layer（03）——可迁移认知（8 KO）
- KO-01 图执行引擎 = 版本化状态机（R4）
- KO-02 状态契约全链路：编译期→运行期（R2）
- KO-03 interrupt/HITL 生命周期协议（R2）
- KO-04 版本不变量：无锁并发的正确性基础（R3）
- KO-05 记忆分层：短时/长时/增量（R4）
- KO-06 失败可恢复性不变量（R3）
- KO-07 可插拔存储接口模式（R1）
- KO-08 增量计算复用（R1）

### Flow Atlas（04）——七类流
Control / State / Data / Evidence / Authority / Memory / Policy——关键 Edge 全部可回溯 `file:symbol`。

### Candidates（05）——7 条未验证假说
含跨项目假说（subgraph 状态隔离、postgres 分布式冲突、v1 迁移中间件）。

### Validation（06）——验证报告
Truth/Coverage/Flow/Abstraction/Counterexample/Epistemic 六审计 + Blind Reconstruction×3，无 CONTRADICTED。

## 3. 关键认知锚点（一句话版）
1. **并发不需要锁**：channel_versions 单调版本 + versions_seen 触发记账 = 无锁乐观并发（KO-04）
2. **interrupt 是"重执行协议"不是"暂停点"**：恢复后节点从头重跑，resume 值按调用顺序匹配（KO-03）
3. **checkpoint 是时间机器**：每超步一快照，thread_id 是主键，time-travel 调试是架构副产品（KO-05）
4. **可插拔一切**：saver/store/serde 三接口解耦，checkpoint-conformance 保证跨实现一致（KO-07）
5. **状态聚合是声明式的**：LastValue/Topic/BinaryOperator/Delta 由 schema 推导，用户不写合并逻辑（KO-01）

## 4. 质量指标
Facts 100+（全部可回溯）→ EK 48（links 96 条，游离 0）→ KO 8（聚合规则 100% 覆盖，平均簇规模 6.0）→ 7 类流 → 7 Candidates。测试揭示行为：test_pregel 9.8k 行 + delta channel 专项测试族 + time-travel 3.9k 行（详见 run_metadata.yaml）。
