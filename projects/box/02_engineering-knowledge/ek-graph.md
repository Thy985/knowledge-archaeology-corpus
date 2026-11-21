# Engineering Knowledge Graph — Strands Box（EK-B01~B20）

> v3.1 规范：每条 EK 声明 links（六类边），KO 从 EK 图簇聚合（R1-R4）。证据格式 `文件:行`（repo HEAD `069c39c`）。

## EK 图节点

### EK-B01 · 策略是唯一授权者（CapabilityOutcome 无 Allow/Deny）
- **机制**：`CapabilityOutcome{Applied(edits), Unavailable(fault)}`——没有 Allow 也没有 Deny，能力只能"应用编辑"或"无法安全执行→什么都不出"（fail-closed）；egress-gateway 不依赖 policy crate
- **证据**：crates/egress-gateway/src/capability/traits.rs:12-50；docs/design/decisions.md:559-570（"Any code path that could return 'allowed' without asking the engine is deleted"）
- **links**: constraint→EK-B03（deny-only floors 只能减）；mechanism↔EK-B02（decider fold 无自身裁决）
- **层**: L2

### EK-B02 · 能力折叠与 mutation 冲突检测
- **机制**：decide_request 把每 capability 的 outcome 折进 Verdict；任一 Unavailable → `Verdict::Deny{Credential}`；**跨 capability 同一 target 的 mutation 冲突 = 配置冲突 → 拒绝**（finish_allow 先 first_collision）
- **证据**：crates/egress-gateway/src/capability/decider.rs:1-60
- **links**: dependency→EK-B07（Zeroizing Mutation 语义）；mechanism↔EK-B01
- **层**: L2

### EK-B03 · deny-only floors 跑在策略之下，永不并列
- **机制**：网络路径 floors 跑在 policy 前、interpreter 路径跑在 policy 后（reachable paths check 只能减不能加）；拒绝 operator home 外/box 目录/本次加载的配置与策略；**不保护凭据**（无 credential floor 在解释器下——scope 每条规则）
- **证据**：architecture.md:216-240（deny-only check runs after the permit）；decisions.md:595-608、608-625
- **links**: constraint→EK-B01（floor 不能授权）；causal→EK-B12（credential 暴露面）
- **层**: L2

### EK-B04 · 单引擎单历史：时间性策略的运行时前提
- **机制**：每 box 恰一个 policy engine、一个历史；跨 enforcement point 共享历史使 fs:read→net deny 可表达；history durable per box（跨 run），无回退，非审计记录
- **证据**：decisions.md:163-177、761-788；policy.md "One engine, and it's the only authorizer"
- **links**: mechanism↔EK-B05（temporal 规则消费同一历史）；subsystem→EK-B06（policy crate facade）
- **层**: L2

### EK-B05 · Dogwood temporal 规则：闭集上下文 + 单一动作
- **机制**：temporal 算子（formerly/within + count）骑在普通规则的闭集上下文上；budget 命名单一动作（不跨 family）；加载期 judge()：**无条件 permit 使 temporal permit 失效 → 拒绝加载**（InertTemporalPermit），heuristic 形状 warn
- **证据**：decisions.md:748-788；crates/policy/src/cap.rs:1-60（judge/finding）
- **links**: dependency→EK-B04；constraint→EK-B20（加载期拒绝）
- **层**: L2

### EK-B06 · 策略 crate 单门面（EffectInterceptor seam）
- **机制**：policy crate 暴露一个 facade；引擎是发布版 dogwood 引擎 pin 精确版本；无第二个 allow path；CapabilityOutcome 类型承载"规则"
- **证据**：decisions.md:889-900（one facade）；crates/policy/src/lib.rs
- **links**: subsystem→EK-B04；constraint→EK-B01
- **层**: L2

### EK-B07 · Zeroizing Mutation：凭据在编辑中不驻留
- **机制**：Mutation 值（header/path/query 注入）用 Zeroizing<String>，Debug 实现 redacted，drop 时擦除；target_key 归一化（header 大小写）用于冲突检测
- **证据**：crates/egress-gateway/src/boundary/verdict.rs:1-60
- **links**: dependency→EK-B09（注入后 response 擦除）；mechanism↔EK-B02
- **层**: L1/L2

### EK-B08 · 凭据边界 = box：占位符换真
- **机制**：env://（启动读一次）与 aws:// SigV4（每请求签名）两 scheme；agent 拿占位符（或占位 AWS keys），gateway 出站换真；response 回显的凭据 redact；"inject = always, against no placeholder" 是 advisory override（警告）
- **证据**：credentials.md 全文；crates/credentials/src/attach.rs:36-40、183、381-384；decisions.md:738（binding is configuration）
- **links**: causal→EK-B09（gateway 注入点）；constraint→EK-B11（box 是凭据边界）
- **层**: L2

### EK-B09 · 终止所有连接：无隧道透明
- **机制**：gateway 终止 TLS（不 splicing/raw copy）；CONNECT 明文/非 HTTP 不可达；net:connect 在 DNS 前+每个 dial 地址再判；cloud-metadata 保护是 policy（forbid 规则）非编译列表
- **证据**：decisions.md:571-595；egress.md "Decides" 表
- **links**: dependency→EK-B08；contrast↔EK-B17（解释器 URL 预检）
- **层**: L2

### EK-B10 · 代理变量强制 + 内核两端口
- **机制**：HTTP(S)_PROXY 等变量由 Box 组合环境强制（box.toml env 表禁设这些名）；NO_PROXY=127.0.0.1,localhost；每 run 内存自签 CA（仅证书落 trust/cert.pem，6 环境变量指向它）；内核限制沙箱进程仅连 gateway+collector 两本地端口（macOS）
- **证据**：egress.md "How traffic reaches the gateway"；docs/design/macos-enforcement.md
- **links**: subsystem→EK-B09；constraint→EK-B18（叶沙箱 contain_egress 例外）
- **层**: L2

### EK-B11 · box 是凭据边界（跨进程粒度）
- **机制**：box 内每个进程拿到每条 route（gateway 无法分辨请求来自哪个进程）→ 不可对单进程隔离凭据；需隔离的进程放独立 box；文档明示
- **证据**：credentials.md "The box is the credential boundary"；decisions.md（box-is-the-credential-boundary）
- **links**: constraint→EK-B08；causal→C-B1（组织级候选）
- **层**: L2

### EK-B12 · 解释器暴露面最宽 + 无凭据 floor
- **机制**：Shell/Monty 可命名 operator home 下任何路径；一个无路径条件的 fs:read permit 读 ~/.ssh ~/.aws；reachable paths floor 不命名这些路径——产品最宽暴露面，明确写入设计文档
- **证据**：policy.md "Reach through the box's trusted process is wide"；decisions.md:608-625
- **links**: causal→EK-B03；contrast↔EK-B16（resolved token 绑定）
- **层**: L2（诚实标注的已知暴露）

### EK-B13 · 解释器运行在可信进程（残余风险明示）
- **机制**：Shell/Monty 在可信进程内（旁 policy 引擎+CA key+已解析密钥）；"mediation rather than memory safety"——解释器内存缺陷=可信进程内代码执行；因无状态跨调用 + 全部 I/O 经决策而可接受
- **证据**：decisions.md:207-262（interpreters brokered aliases / no state crosses call）；architecture.md:96-100
- **links**: mechanism↔EK-B14（broker 三腿）；subsystem→EK-B06
- **层**: L2

### EK-B14 · 无状态跨调用 + framed/versioned 协议
- **机制**：解释器不跨调用保状态（一命令导出的变量下一命令 unset）；broker 协议 framed/versioned（PROTOCOL_VERSION），拒绝带 reason code；alias 无权力
- **证据**：decisions.md:262-276；architecture.md "The alias"
- **links**: mechanism↔EK-B13；constraint→EK-B03
- **层**: L1/L2

### EK-B15 · trampoline：单向遏制模式
- **机制**：macOS sandbox 单向不可撤销 → run 从不自调用 sandbox；起专用 trampoline（自遏制后 exec 目标程序）；配置以继承 descriptor 跨 exec 边界，digest 绑定；trampoline 零决策
- **证据**：decisions.md:1019-1039；architecture.md "The trampoline"；crates/containment/src/backend/macos/
- **links**: mechanism↔EK-B19（沙箱 grants 一次构建固定）；constraint→EK-B10
- **层**: L2（可迁移模式）

### EK-B16 · 先解析后判定 + resolved token 绑定对象
- **机制**：Shell 先解析命令/路径（跟 symlink），对解析结果判定（shell:exec on resolved command / fs:* on resolved path），再对解析到的对象执行——决策与效果之间换 symlink 不能分流
- **证据**：architecture.md "The path is resolved before it is judged"；decisions.md:207（resolved-token-binds-the-object-it-names）
- **links**: constraint→EK-B03；contrast↔EK-B12
- **层**: L2

### EK-B17 · 解释器 URL 预检（net 双拒绝）
- **机制**：Shell 内 curl / Monty fetch() 先拒非 http(s) URL、localhost/字面 loopback/private/link-local 地址，再走 gateway（只信 box CA）
- **证据**：egress.md "Box's own interpreters use it too"
- **links**: contrast↔EK-B09；subsystem→EK-B13
- **层**: L1/L2

### EK-B18 · 叶沙箱：host 二进制/stdio MCP 各自遏制
- **机制**：工具/本地 MCP server 各跑独立叶沙箱（grants 来自自身表，无 alias 无 socket 路）；shell:spawn permit 才启动；contain_egress=false 是声明的叶逃逸（残余风险）
- **证据**：decisions.md:290-322、462-497；architecture.md "A tool's sandbox"
- **links**: constraint→EK-B10；mechanism↔EK-B19
- **层**: L2

### EK-B19 · 沙箱 grants 一次构建、终身固定 + 启动披露
- **机制**：run 构建 grants 不运行程序、box 生命内固定；每直接 fs grant 启动时 stderr 披露（无 fs:* 决策覆盖它）；未判定的 grant 拒绝 box
- **证据**：decisions.md:944-956、371-408；architecture.md step 6
- **links**: mechanism↔EK-B15；constraint→EK-B03
- **层**: L2

### EK-B20 · 加载期规则判定：不能触发即拒
- **机制**：policy 装时解析+验证，失败拒 box；"rule that cannot fire is refused at load, a rule that might warns"（cap.rs judge 实现在 inert temporal permit 上）
- **证据**：decisions.md:679-704；crates/policy/src/cap.rs
- **links**: constraint→EK-B05；subsystem→EK-B06
- **层**: L2

## 图指标
- EK 节点 20（B01~B20）；links 覆盖率 100%；平均出边 ~1.7；无孤立节点（全部连图）
- 与 corpus 连接节点：dogwood（B05 语言）、MXC（B01/B03/B08 对照）、dsh（B14/B18 应用层 seam 对照）
