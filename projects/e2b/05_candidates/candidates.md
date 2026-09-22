# 05 Candidates — E2B（未确认内容，严格与 KO 区分）

> 以下内容均不能确认，留在 Candidate。标注：类型 / 当前证据 / 缺失证据 / 验证路径。

---

## C-01 签名 URL 防重放窗口（Scope-uncertain）
- **类型**: Tentative pattern / scope-uncertain
- **当前证据**: signature.ts 公式含 `[:expiration]` 可选过期段；`v1_` 前缀 base64（sandbox/signature.ts）
- **缺失证据**: 过期段的具体语义（时长、服务端校验）、重放窗口大小、与 trafficAccessToken 的关系未在代码内完整读到（signature.ts 未全文精读）
- **验证路径**: 精读 signature.ts + spec/openapi.yml 对应端点描述
- **状态**: Hypothesis（不升 KO）

## C-02 inflight slot 提前释放的实际影响（Unresolved）
- **类型**: Unresolved contradiction（代码自陈 TODO）
- **当前证据**: inflight.ts 注释自陈"slot 在响应头到达即释放而非 body 消费完"
- **缺失证据**: 该行为在高并发流式场景下的实测影响（连接复用窗口 vs 超限风险）；是否已触发过真实问题
- **验证路径**: 仓库 issue 检索 / 基准测试
- **状态**: Observation（TODO 事实确定，影响未确定）

## C-03 服务端生命周期默认值（keepMemory/autoResume/onTimeout 默认）
- **类型**: Scope-uncertain
- **当前证据**: SDK 刻意省略未配置字段，"leaves the timeout action to the API"（sandboxApi.ts:1655, 1679）——API 默认值决定行为
- **缺失证据**: 服务端默认值（docstring 称 onTimeout 默认 kill、keepMemory 默认 enabled，但均为"currently"措辞）
- **验证路径**: spec/openapi.yml NewSandboxV2 schema + 服务端文档
- **状态**: Observation

## C-04 Python SDK 与 JS SDK 行为一致性程度
- **类型**: Coverage-uncertain
- **当前证据**: python-sdk 存在 sync/async 双实现；CLAUDE.md 强制等效变更；JS 侧已逐行精读（client/connectionConfig/sandboxApi/errors/envd/rpc/commands/iam/retry/inflight/http2）
- **缺失证据**: Python 侧逐行对照（本次考古未精读 python-sdk 实现体）
- **验证路径**: 后续 refresh run 聚焦 python-sdk + code-interpreter-python
- **状态**: Hypothesis（"双实现等效"是治理承诺，非已验证事实）

## C-05 托管云服务实现（api.e2b.dev 服务器侧）
- **类型**: Scope-uncertain / 仓库边界
- **当前证据**: 本仓库不含服务器；控制面契约在 spec/openapi.yml；e2b-dev/runtime 为 envd 源头（pin f5dc6426）
- **缺失证据**: 服务器侧实现（调度、隔离、计费、配额）
- **验证路径**: 单独考古 e2b-dev/runtime（refresh 或新 job）
- **状态**: Fact（仓库边界确定）— 服务实现内容未知

## C-06 跨项目模式：沙箱平台共享认知（Cross-project hypothesis）
- **类型**: Cross-project hypothesis
- **当前证据**: 本项目中 KO-01（秘密不落地）、KO-03（探测后归因）、KO-05（fail-closed 网络面）有强单项目证据
- **缺失证据**: 其他沙箱/远端执行平台（browser-use 的浏览器沙箱、letta-code 的 agent 运行时、dsh-pentest 的安全测试沙箱）是否共享同构
- **验证路径**: 与 corpus 中既有项目（browser-use/letta-code/mem0）做交叉对照（Benchmark 学习价值点）
- **状态**: Hypothesis —— **禁止写成已验证 Principle**

## C-07 MCP gateway token 文件权限模型
- **类型**: Coverage-uncertain
- **当前证据**: GATEWAY_ACCESS_TOKEN + `/etc/mcp-gateway/.token`（sandbox/index.ts mcp 相关段落提及）
- **缺失证据**: token 文件权限（谁可读）、轮换策略、与沙箱内用户可见性
- **验证路径**: 精读 sandbox/index.ts mcp 段 + mcp-server.json
- **状态**: Observation

---

## 候选与 KO 边界声明

- C-01/C-02/C-03/C-07：单项目内部未决细节，即使未来确认也不改变 KO 层级（仍为工程事实）
- C-04/C-05：覆盖边界声明（本次考古范围 = js-sdk 深读 + python 结构对照；服务端与 python 实现体未深读）
- C-06：唯一的跨项目候选——只有经 corpus 交叉验证后才可升 L4/L5，目前严格保持 Hypothesis
