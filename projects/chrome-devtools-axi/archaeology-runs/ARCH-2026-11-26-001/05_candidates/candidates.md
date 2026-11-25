# 05 · Candidates — 未确认、跨项目、暂定模式

> 候选区不是结论区。每条都显式标注 `type`、`confidence`、`open_question`、`how_to_test`。任何 `cross_project_candidate` 都不得被下游当成已验证 Principle。

## C-01 · 同机不同 UID 能否访问 loopback bridge（未测）

```yaml
id: C-01
type: hypothesis
confidence: low
scope: "多用户 Unix 主机 / 共享 CI runner"
claim: "bridge 只 bind 127.0.0.1 且无 token；同机其他 UID 若可建立 loopback TCP，就能调用 /call。"
why_open: "本轮没有启动 bridge，也没有跨 UID 运行时测试；OS 套接字权限、防火墙、容器网络命名空间都可能阻断。"
evidence: [EV-015, EV-017, EV-018]
how_to_test: "在一次性 VM 创建两个 UID，A 启动 bridge，B curl 127.0.0.1:9224/call；记录成功/拒绝与 OS 版本。"
risk_if_true: "高：本地多租户/共享 runner 的控制面横向越权。"
status: open
```

## C-02 · `isBridgeProcess` 子串身份可伪造 / PID 复用（未测）

```yaml
id: C-02
type: hypothesis
confidence: medium
scope: "terminateBridgeProcess 的组杀前置检查"
claim: "`ps -p <pid> -o command=` 的子串匹配 `chrome-devtools-axi-bridge` 可能被同名进程伪造；PID 复用也可能让旧 PID 文件指向无关进程。"
why_open: "代码只做子串匹配（src/client.ts:269-279）；没有 inode/start-time/cgroup 绑定；未做运行时伪造测试。"
evidence: [EV-007, EV-008]
how_to_test: "伪造 argv[0] 含 bridge 名的进程并写入 bridge.pid；观察 ensureBridge 是否组杀。"
risk_if_true: "中：误杀同组进程或拒绝服务；需要本地写权限才能利用。"
status: open
```

## C-03 · 状态文件权限与隐私（未测）

```yaml
id: C-03
type: uncertain_pattern
confidence: low
scope: "~/.chrome-devtools-axi/ 下的 bridge.pid / selected-page-id / snapshot-generation"
claim: "状态目录默认权限由 umask 决定，可能对其他本地用户可读；selected page id 与 PID/port 不是秘密，但会泄露会话结构。"
why_open: "没有检查 mkdir 的 mode 参数；没有实测文件权限；没有多用户环境。"
evidence: [EV-003, EV-011, EV-027]
how_to_test: "umask 000 下启动，ls -l 状态目录；检查是否 0700/0600。"
risk_if_true: "低-中：信息泄露 + 本地进程更容易定位 bridge。"
status: open
```

## C-04 · legacy uid 不绑定原始 epoch，削弱 per-ref 安全边界

```yaml
id: C-04
type: uncertain_pattern
confidence: medium
scope: "旧调用方未使用 gN: 前缀的 uid"
claim: "`checkUidGeneration('@5', 7)` 返回 stale:false；legacy ref 不被拒绝，只受当前页面 generation/mutations 约束。它可以指向同一页面上后来的不同元素。"
why_open: "这是明确的兼容性例外（EK-17）；没有构造一个跨 snapshot 的 legacy ref 攻击来证明可达性。"
evidence: [EV-033, EV-035, EV-036]
how_to_test: "连续两次 snapshot，用第一次的 legacy uid 在第二次之后执行 click；观察是否命中错误元素或 STALE_REF。"
risk_if_true: "中：旧客户端可能误操作；新客户端不受影响。"
status: open
```

## C-05 · 重复 Host / HTTP2 `:authority` / 代理改写未测

```yaml
id: C-05
type: hypothesis
confidence: low
scope: "Host/Origin gate 的 HTTP 解析层"
claim: "Node 对重复 Host 头的取舍、HTTP/2 `:authority`、反向代理改写可能让 `extractHostHeaderHostname` 看到与客户端意图不同的值。"
why_open: "测试覆盖 `[::1]evil.com` / `localhost.evil.com`，但没有重复 Host、没有 h2、没有代理。"
evidence: [EV-015, EV-047]
how_to_test: "用原始 socket 发两个 Host 头；用 h2 客户端发 `:authority`；用 nginx 反代。"
risk_if_true: "中：gate 可能被解析差异绕过；取决于部署。"
status: open
```

## C-06 · `roots` 任意绝对路径放大文件访问边界

```yaml
id: C-06
type: hypothesis
confidence: medium
scope: "POST /call 的 roots + collectRootDirs"
claim: "调用方可把 `/`、`$HOME` 或任意祖先目录设为 MCP roots；`collectRootDirs` 还会把输出路径的最近存在祖先加入。若 MCP 的 validatePath 以 roots 为允许范围，这会把文件读写扩展到调用方 cwd 之外。"
why_open: "代码只校验数组 + 绝对路径（src/bridge.ts:428-441）；没有在 live MCP 上验证实际可写范围。"
evidence: [EV-019, EV-024, EV-025]
how_to_test: "启动真实 bridge + chrome-devtools-mcp，调用 screenshot --filename /tmp/outside.png 并传 roots=['/']；检查是否写入。"
risk_if_true: "高：任意本地路径写/读，取决于下游工具。"
status: open
```

## C-07 · Keychain 隔离只有代码/测试证据，无 live Chrome 证据

```yaml
id: C-07
type: evidence_gap
confidence: high
scope: "launch 模式的 --use-mock-keychain / --password-store=basic"
claim: "隔离 flag 的代码路径与 10 个矩阵测试存在，但本轮没有启动真实 Chrome，无法确认实际 keychain 不被读写。"
why_open: "无 GUI Chrome / macOS Keychain 环境；只能验证 argv 构造。"
evidence: [EV-020, EV-021, EV-045]
how_to_test: "macOS 上 launch --isolated，检查 login keychain 是否新增条目；attach 模式对照。"
risk_if_true: "若 false negative：隔离承诺不成立；若 true：符合设计。"
status: open
```

## C-08 · 多 session attach 同一浏览器的 cross-talk

```yaml
id: C-08
type: known_limitation
confidence: high
scope: "两个 session 都 attach 到同一个外部 Chrome"
claim: "per-session generation 不能防止浏览器状态 cross-talk；cookie/storage/tab 共享。"
why_open: "README/AGENTS 明确声明；没有运行时演示两个 session 互相影响。"
evidence: [EV-001, EV-045]
how_to_test: "两个 session attach 同一 browser，session A 登录，session B 读 cookie。"
risk_if_true: "中：隔离预期错配；不是代码 bug，是 scope 声明。"
status: open
```

## C-09 · `run` / `eval` 的脚本权限边界

```yaml
id: C-09
type: known_limitation
confidence: high
scope: "run 命令的临时 .mjs 动态 import"
claim: "run 脚本与 CLI 同权限，可读文件、发网络请求、读环境变量；不是沙箱。"
why_open: "代码注释/文档说明；没有实际越权演示。"
evidence: [EV-040]
how_to_test: "run 脚本读 /etc/passwd 或 process.env，确认成功。"
risk_if_true: "设计如此；对不可信脚本是高风险。"
status: open
```

## C-10 · `npx -y chrome-devtools-mcp@latest` 的供应链/可复现性

```yaml
id: C-10
type: hypothesis
confidence: medium
scope: "本地 stdio fallback"
claim: "最后一档 fallback 拉取 latest，未 pin 版本；不同时间/网络可执行不同上游代码，且 npx 供应链风险由 npm registry 决定。"
why_open: "代码/测试验证 fallback 顺序；没有 SBOM/签名验证。"
evidence: [EV-022, EV-023]
how_to_test: "断网/指定 registry，观察 fallback；对比两次 latest 解析的版本。"
risk_if_true: "中：不可复现 + 供应链。"
status: open
```

## C-11 · 拒绝日志的注入/伪造与无 per-caller 归属

```yaml
id: C-11
type: uncertain_pattern
confidence: low
scope: "logForbidden 的 host/origin/method/url 拼接"
claim: "拒绝日志把客户端可控的 Host/Origin/URL 拼进字符串；若日志被当结构化数据消费，可能注入换行/伪造字段。且没有 caller 身份，无法归因。"
why_open: "没有检查日志实现是否转义；没有实测恶意 Host 值。"
evidence: [EV-016, EV-036]
how_to_test: "发送含换行/制表符的 Host，观察日志行数。"
risk_if_true: "低：日志污染/审计误导。"
status: open
```

## C-12 · 本地控制面暴露分类（跨项目候选）

```yaml
id: C-12
type: cross_project_candidate
confidence: low
scope: "KnowlegeMap 2026-11-26 的 7 模式卡 + 本项目证据"
claim: "本地控制面可按“谁能触发”分为：同机进程、浏览器页面、扩展、代理、容器、IDE 插件、共享 runner。不同触发者的认证需求不同；Host/Origin 只覆盖浏览器页面一类。"
why_open: "这是跨对象归纳，不是本项目已验证事实；7 模式卡是问题定义，不是代码证据。"
evidence: [EV-015, EV-017, EV-018, EV-019]
how_to_test: "对每个触发者做最小权限实验；建立统一威胁矩阵。"
risk_if_true: "若成立：可作为跨项目 Pattern；当前只能作 Candidate。"
status: open
```

## 候选统计

- 总数：12
- hypothesis：5（C-01/C-02/C-05/C-06/C-10）
- uncertain_pattern：3（C-03/C-04/C-11）
- known_limitation：2（C-08/C-09）
- evidence_gap：1（C-07）
- cross_project_candidate：1（C-12）

> 类型计数以每条 YAML 的 `type` 为准；不要用本段自然语言覆盖。
