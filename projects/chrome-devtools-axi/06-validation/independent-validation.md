# 06 · Independent Validation — Blind Reconstruction

> **方法声明**：本轮由同一 MainAgent 在只读仓库上做第二次独立走读，先写 Independent Findings，再与 Package 对比。不是独立模型/独立进程，因此不能声称“完全盲”；但所有 Independent Findings 均重新落到 `src/`、`test/`、workflow 的实际行号，未把 Package 的 KO 文本当作事实来源。Package 的原始产物未被修改；修正写入 `06-validation/reconciliation.md` 和 `03-knowledge-layer` 的版本化更新。

## 1. Independent Findings（先于对比）

| ID | 独立发现 | 证据 |
|---|---|---|
| IF-01 | CLI 每次执行短命进程，跨命令会话由 detached bridge 持有 | `README.md:117-160`；`src/client.ts:352-370` |
| IF-02 | bridge 在 `127.0.0.1` 暴露 `/health`、`/health?deep=1`、`/tools`、`/call` | `src/bridge.ts:6-9, 1225` |
| IF-03 | Host 必须 loopback；Origin 缺失允许、存在必须 loopback；gate 在路由前 | `src/bridge.ts:217-320, 531-546` |
| IF-04 | `/call` 无 token/auth，`name` 直接 `client.callTool` | `src/bridge.ts:476-520, 610-620`；commit `4ee5a49` |
| IF-05 | `roots` 来自请求体，只校验数组 + 绝对路径；`collectRootDirs` 加 cwd/祖先 | `src/bridge.ts:428-441, 1098-1155`；`src/client.ts:640-690` |
| IF-06 | 复用前 deep health；失败 terminate + clear selection + spawn | `src/client.ts:473-530`；`src/bridge.ts:557-580` |
| IF-07 | selected page 只由显式 select/new 写；`list_pages` 的 `[selected]` 是展示 | `src/selected-page.ts:1-20, 109-125`；`src/pages.ts:1-20` |
| IF-08 | reconnect 是 identity epoch；显式 pageId 返回 identity error，不 retarget | `src/bridge.ts:360-410, 470-520` |
| IF-09 | generation + MutationObserver 联合判定 freshness；stale 抛错 | `src/uid-freshness.ts:45-60, 118-140`；`src/generation.ts:23-60` |
| IF-10 | keychain flag 只在 launch 分支追加；attach 不加 | `src/bridge.ts:683-760`；`test/keychain-isolation.test.ts` |
| IF-11 | session 只隔离 bridge；多 session attach 同一浏览器会 cross-talk | `README.md` keychain 段；`AGENTS.md` invariants |
| IF-12 | MCP `isError:true` 转 `{error}`，CLI 非零退出 | `src/bridge.ts:330-350`；`src/client.ts:790-850` |
| IF-13 | 测试：29 文件、654 静态 `it(`、796 执行用例 | 本地 `pnpm test`；`rg -c 'it\\(' test/*.test.ts` |
| IF-14 | CI 在 Node 24 跑 frozen install/build/test；airlock 用 pnpm lint | `.github/workflows/ci.yml`；`.airlock/lint.sh` |
| IF-15 | `isBridgeProcess` 只做 `ps` 子串匹配，无 start-time/inode 绑定 | `src/client.ts:269-279` |
| IF-16 | legacy uid 无 generation 时 `stale:false`，但仍过页面状态门 | `src/uid-freshness.ts:118-140`；`test/snapshot.test.ts:200-240` |

## 2. 对比结果（按 KO）

| KO | 判定 | 理由 | 动作 |
|---|---|---|---|
| KO-01 三份连续性 | CONFIRMED（项目级） | 三条机制独立存在，且都在 `ensureBridge`/selection/generation 的同一跨进程状态底座上 | 保留 L4，但 scope 收紧为“同一 bridge + 同一 session” |
| KO-02 origin gate ≠ caller auth | PARTIALLY_CONFIRMED | gate、无 auth、任意 roots、无 per-caller 审计均为 Fact；“任何同机进程”未运行时验证，且多 UID/容器/防火墙会改变可达性 | scope 从“任何同机进程”降为“能建立 loopback 连接并可设置 Host 的本地调用方”；`any` 改为 `a local caller with loopback reachability` |
| KO-03 custody-scoped policy | DOWNGRADED（L4 → L3） | 只有 session 与 keychain 两个实例，且一个是 bridge 隔离、一个是浏览器 flag；不足以支撑 L4 认知模型 | 改为 L3 项目级 Pattern，标题保留，claim 限定“本项目的两个隔离面” |
| KO-04 identity epoch fail-closed | CONFIRMED | 代码、测试、错误码闭合；未发现自动 retarget 路径 | 保留 |
| KO-05 freshness = generation + quiescence | PARTIALLY_CONFIRMED | 主路径闭合；legacy ref 例外削弱“所有 ref 都绑定 epoch”；无 live DOM 测试 | 保留 L4，但 claim/scope 显式写入 legacy 例外与“无 live DOM” |
| KO-06 多触发 lifecycle | CONFIRMED（项目级） | deep health / onclose / idle / group kill 四条独立路径都有代码与测试 | 保留 L3 |
| KO-07 显式 caller context | CONFIRMED（项目级） | cwd/deadline/callable/stdout 四个上下文均有 chokepoint 与测试 | 保留 L3 |
| KO-08 质量/发布 custody | OVER_GENERALIZED（scope 收紧） | 结论是正确的工程常识，但只来自一个仓库；没有跨项目证据，novelty 低 | 保留 L3 但 scope 限定“本仓库的 release/CI 治理”，不得跨项目推广 |

## 2.5 Validator 判决（六类）

| Validator | 判决 | 依据 |
|---|---|---|
| truth-auditor | PARTIAL | 4 KO CONFIRMED；2 PARTIALLY（scope/legacy 例外）；0 CONTRADICTED |
| coverage-auditor | FAIL | 7 个 MISSING：跨 UID、HTTP2/重复 Host、状态文件 mode、live roots、PID 伪造、live keychain、npx 供应链 |
| flow-auditor | PASS（静态） | 48 条 Edge 的 source/target/symbol/condition 均可回到代码或 workflow；无 `INCORRECT` |
| abstraction-auditor | PARTIAL | KO-03 L4→L3；KO-08 标 OVER-ABSTRACTED 并收紧 scope；其余 L4 通过 |
| counterexample-hunter | PASS | 8 个 KO 各 3 次定向攻击，共 24 次；未发现可推翻 claim 的路径，但发现 2 个 scope 膨胀 |
| epistemic-auditor | PASS | 无 Fact→Principle 跃迁；无 L5；legacy/Candidate 未被写成 Validated Pattern |

## 3. 三个成功

1. **S-01 Host/Origin gate 的真实边界被正确识别**：不仅有 gate，还能指出它只覆盖浏览器侧、`/call` 无 auth。`src/bridge.ts:531-546` + commit `4ee5a49` 闭合。
2. **S-02 identity epoch 的 fail-closed 行为被正确识别**：`didMcpPageIdentityChange`、`PAGE_IDENTITY_CHANGED_ERROR`、`clearSelectedPageId` 三点一致，没有被“自动恢复 tab 更方便”带偏。
3. **S-03 freshness 不是时间戳**：`generation` + `mutations` 的联合谓词、`STALE_REF` 错误、legacy 例外都被准确记录，没有把 legacy 说成“所有 ref 都安全”。

## 4. 三个错误

1. **E-01 scope 膨胀**：KO-02 原文“任何同机进程都可以”是超出证据的强断言。代码证明“无 auth + 任意 roots + 无审计”，不证明“任意 UID/任意网络命名空间均可达”。已收紧。
2. **E-02 单仓库升维**：KO-08 从“这个仓库的 CI/guard/release”升成“质量/发布声明必须由机器强制 custody”，没有跨项目样本；这是正确但低 novel 的常识。已收紧 scope，保留 L3 但标 OVER_GENERALIZED。
3. **E-03 两个实例升 L4**：KO-03 的 custody 机制只有 session/keychain 两个面，且不是同一子系统的两个独立实现；L4 认知模型过强。已降 L3。

## 5. 遗漏（MISSING）

| ID | 遗漏 | 影响 |
|---|---|---|
| M-01 | 未测同机不同 UID 的 loopback 可达性 | KO-02 的“本地 caller”只能停在代码层 |
| M-02 | 未测重复 `Host` / HTTP2 `:authority` / 反向代理改写 | gate 解析边界不完整 |
| M-03 | 未测状态目录/文件 mode | C-03 无法升级为事实 |
| M-04 | 未测真实 MCP 的 roots 文件访问范围 | C-06 无法量化 |
| M-05 | 未测 `isBridgeProcess` 伪造/PID 复用 | C-02 保持 hypothesis |
| M-06 | 无 live Chrome，keychain 隔离只有 argv/测试证据 | C-07 保持 evidence_gap |
| M-07 | 未验证 `npx -y ...@latest` 的实际版本与供应链 | C-10 保持 hypothesis |

## 6. Counterexample 攻击记录

- **攻击 KO-01**：只检查 PID 就复用 bridge；代码会在 deep health 失败时 respawn，但若 CDP target 假活（list_pages 成功但页面已换），三份连续性仍可能错。KO-01 保留，但 scope 加“deep health 本身的假阳性未排除”。
- **攻击 KO-02**：`/call` 无 auth 是事实；但“同机进程”需要能连 loopback。容器/网络命名空间/防火墙可阻断。KO-02 降为 PARTIALLY。
- **攻击 KO-03**：attach 模式加 keychain flag 是否更安全？测试证明 flag 只在 launch 追加；attach 是用户浏览器，工具无权替用户改 keychain。KO-03 仍成立，但只到 L3。
- **攻击 KO-04**：从 `list_pages` 的 `[selected]` 重建 id；代码注释/parser 明确拒绝。KO-04 成立。
- **攻击 KO-05**：用第一次 snapshot 的 legacy uid 在第二次后点击；legacy 只受当前状态门，可能指向同页不同元素。KO-05 降为 PARTIALLY，C-04 保留。
- **攻击 KO-06**：只查 PID；deep health/onclose/idle/group kill 四条路径分别覆盖。KO-06 成立。
- **攻击 KO-07**：相对输出路径按 bridge cwd；`resolveOutputPath` 按 CLI cwd。KO-07 成立。
- **攻击 KO-08**：手改 CHANGELOG 绕过 release；guard-generated-files 检查 base...head。KO-08 只到 L2，不能当跨项目原则。

## 7. Benchmark Case

**BC-2026-11-26-A：本地控制面 6 问**

| 问题 | 本项目答案 | 分数 |
|---|---|---|
| Q1 谁能触发 | 浏览器页面（被 Host/Origin 挡）+ 本地 loopback 调用方（无 auth） | 1/1 |
| Q2 调用方身份 | 无 token/peer credential；`/call` 直接映射工具 | 1/1 |
| Q3 权限范围 | 任意 MCP tool + 任意绝对 roots；无 allowlist | 1/1 |
| Q4 证据/审计 | 仅拒绝日志；无 per-caller、无 tool 参数审计 | 1/1 |
| Q5 失败是否响亮 | `isError` → `{error}` → 非零退出；stale/identity 错误明确 | 1/1 |
| Q6 部署差异 | 多 UID/容器/HTTP2/代理未测 | 0/1 |
| **总分** |  | **5/6** |

> 分数只用于本轮 benchmark，不是项目安全评级。Q6 的缺失正是本轮的 MISSING 集合。

## 8. 判定统计

- CONFIRMED：4（KO-01/04/06/07）
- PARTIALLY_CONFIRMED：2（KO-02/05）
- DOWNGRADED：1（KO-03）
- OVER_GENERALIZED：1（KO-08）
- MISSING：7 个证据缺口（M-01..M-07）
- CONTRADICTED：0
- NEEDS_HUMAN_REVIEW：0 个 KO；但 C-01/C-02/C-06 建议人工复核
- **Overall：PARTIALLY_CONFIRMED**
- **Validation Gate：PARTIAL**（静态 + 本机测试通过；无 live Chrome、无运行时攻击、无跨 UID）

## 9. 限制

本轮不是独立模型/独立进程，不能消除确认偏差。所有“运行时未执行”的项都保留在 Candidates/MISSING，不用“代码看起来对”替代运行时证据。下一次 refresh 应优先补 M-01、M-02、M-04、M-05。
