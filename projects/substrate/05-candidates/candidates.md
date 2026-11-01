# Candidates — Agent Substrate（05 未验证假说）

> 不能确认的内容留在候选。Hypothesis 明确标记，绝不冒充 Fact/Principle。

## C-S01（Hypothesis）snapshot 三级保真在实际系统中是否全部可达
- 断言：MEMORY 级快照在 gVisor 与 microVM 两端均可用且保真一致
- 当前证据：[doc] architecture.md Sandbox Classes 声明 + [code] checkpoint.go 契约注释（VOLUMES/ROOTFS/MEMORY 分支）
- 缺失证据：未运行实测恢复保真（benchmarking/ 提供路径未执行）；架构文档自述 aspirational
- 验证路径：internal/e2e + benchmarking 实测 memory restore；gVisor --allow-connected-on-save 是否所有 runsc 版本可用

## C-S02（Hypothesis）500ms/1500+ 性能声明成立且为稳态值
- 断言：README <400ms resume / 1500+ activations/sec 为可复现实测（而非路线图）
- 当前证据：[doc] README 声明；benchmarking/（Locust + nighthawk + telemetry 量测）存在
- 缺失证据：本轮未跑 benchmark；North Star（100ms/1b actors）明确为目标
- 验证路径：benchmarking/deploy_locust.sh 按 README 部署实测；需 ate-system 集群

## C-S03（Hypothesis）请求停车的实现与文档一致（lot 1024/budget 5s）
- 断言：ext_proc 停车、lot 容量、budget 回退真实生效
- 当前证据：[doc] request-parking.md 详细契约 + 配置 flag 名
- 缺失证据：未采样 cmd/atenet/internal/router 中 parking 实现代码
- 验证路径：采样 extproc/ 与 resumer 代码；e2e 注入饱和场景

## C-S04（Hypothesis）"凭据注入后丢弃"在网关实现中真不可读取
- 断言：egress 网关代码路径上原始 secret 不可被读取/外泄（非仅文档承诺）
- 当前证据：[doc] egress-credential-injection.md；[code] cmd/atenet/internal/router/egress/credentials.go 存在（未精读）
- 缺失证据：credentials.go 实现细节（是否真的只透传不落日志）
- 验证路径：精读 credentials.go + 搜索日志路径对 credential 的处理；威胁模型 T-snapshot 类

## C-S05（跨项目假说，来自 KO-S05）凭据外注模式三仓收敛
- 断言：Substrate（egress 注入后丢弃）+ dapr-agents（凭证外置）+ MXC/Strands Box（OS 级隔离 + 凭据外挂）共享同一安全模式："沙箱内永存凭据是首要缺陷，凭据必须外注"
- 当前证据：Substrate [doc/code]；dapr-agents（ARCH-2026-10-25）+ MXC（10-24）+ Strands Box（10-23）corpus 记录
- 缺失证据：三项目同机制的系统性对照研究未做；Substrate 侧仅文档级
- 验证路径：三仓 EK 并排对照（candidate 留待 Evaluation 周）

## C-S06（Hypothesis）atelet/worker 互不透明 + Pod 生命周期解耦在 v1.0.0 后保持
- 断言：AGENTS.md 声明的"仅共享 ateworkerpb+nodepath"边界在演进中不破（当前 v0.6.0 未强制）
- 当前证据：[code] AGENTS.md 声明 "v1.0.0 前不强制"
- 缺失证据：v1.0.0 后契约（无法从当前仓库证明未来）
- 验证路径：跟踪后续 release；对照接口演进

## C-S07（Hypothesis）OpenFGA 授权覆盖所有 50+ RPC
- 断言：每个 Control RPC 都过鉴权（无漏检路径）
- 当前证据：[doc] authorization.md "Every RPC is authorized" + [code] internal/ateapiauth + interceptor 存在
- 缺失证据：未逐一核对 50+ RPC 的检查表（authorization.md 提到 known gaps）
- 验证路径：逐一对照 RPC→check；authorization.md "known gaps" 清单
