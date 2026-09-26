# Independent Validation Report — ARCH-2026-09-22-001 (browser-use)

> Auditor 模式：Blind Reconstruction——先不读原考古结果，独立重读仓库建立 Independent Findings，再与原考古结果对比。
> 本次 Auditor 独立抽查了 6 个代码区域（主循环/captcha 等待/loop detector nudge/SecurityWatchdog 域检查/multi_act 防护/IP 阻断）。

## 1. 判定统计

| 判定 | 数量 | 内容 |
|---|---|---|
| CONFIRMED | 8 | EK-01 主循环四阶段 / EK-02 multi_act 双层防护 / EK-03 软循环检测 / EK-08 三层域检查 / EK-10 captcha 等待 / EK-12 LLM 工厂 / EK-14 watchdog 体系 / EK-21 停止检查 |
| PARTIALLY_CONFIRMED | 3 | EK-11（下载判定机制确认，但"注册回调"细节未完全核对）/ EK-16（商业策略确认，但"AGENTS.md 明文推荐"需核对原文）/ EK-18（云浏览器确认存在，但实际行为未验证） |
| DOWNGRADED | 1 | KO-09：原标 L4 Cognitive Model → 保持 L4 但强制标 `Cross-project validation pending`（单项目证据，不能当 Principle） |
| OVER_GENERALIZED | 1 | C-01 "软门控/硬门控分工是跨项目模式"——原写为较肯定的模式描述，收紧为 Hypothesis 表述 |
| MISSING | 3 | M-1：`block_ip_addresses` IP 绕过防护（security_watchdog.py:209 + profile.py:636）；M-2：captcha 等待的 vendor/outcome 三态细节（success/failed/timeout，service.py:1043-1055）；M-3：loop nudge 的 escalating 分级（≥5/≥8/≥12 不同强度 + "有进展就继续"防误报，views.py:211-230） |
| CONTRADICTED | 0 | 无考古结果与代码矛盾的案例 |
| NEEDS_HUMAN_REVIEW | 1 | EK-15（beta Rust SDK）：原考古未深读 beta 全貌，C-04 已保留 Observation；建议人工复核 beta 价值后再决定是否升 L3 |

## 2. 原 Archaeology 最重要的 3 个成功

1. **软门控/硬门控分工（EK-03 + EK-04 → KO-03/KO-07/KO-09）**：抓住了 browser-use 最独特的工程哲学——循环检测"从不阻断动作"（docstring 原文验证），并把它与预算硬门控对照，抽象出"行为质量靠说服、资源终止靠强制"的稳定关系。这是本项目最有迁移价值的认知。
2. **上下文预算分级降级（EK-04 → KO-02）**：把 compaction→75% 警告→最后一步强制 done→失败强制 done 的四级降级完整还原，证据链闭合（views.py:35-58 + service.py:1542-1593），是"长时 agent 资源管理"的可操作答案。
3. **事件驱动 watchdog 架构（EK-14 → KO-04）**：16 个 watchdog 不是零散功能，而是"横切关注点独立成监视器"的统一架构决策；EventBus 证据（watchdog_base.py + session.py）支撑，主循环保持干净——这是浏览器 agent 基础设施层的设计模式。

## 3. 最重要的 3 个错误

1. **EK-08 域级安全边界不完整（MISSING）**：原考古只写了"三层域白名单检查"（导航前/重定向/新标签），**遗漏了 `block_ip_addresses` IP 编码绕过防护**——WHATWG host canonicalization 镜像（decimal/hex/octal/short-form/percent-encoded/Unicode digits）是安全设计的点睛之笔（绕域名白名单最简单的方式就是 IP 直连），不补这条 EK 会误导读者低估域白名单的实际防御深度。**影响：高**——安全类知识的 coverage 缺口。
2. **EK-03 循环检测的 escalating 细节缺失（MISSING）**：原写"滚动窗口 20 + 页面指纹"，未写 nudge 是**分级升级**的（≥5/≥8/≥12 三档不同强度、且明示"如果每次重复有进展就继续"）。这个"防误报"设计（避免把"有效的重复操作"误判为循环）是软门控的精华——原文丢失了。**影响：中**。
3. **EK-10 captcha 等待细节不精确（MISSING）**：原写"带超时阻塞等待"，未写 vendor（供应商）、outcome 三态（success/failed/timeout）、duration 注入上下文、以及 captcha 等待时间从 step 计时中扣除（step_start_time 重置）。这些细节证明"等待结果必须成为 LLM 可见的 ActionResult"。**影响：低-中**。

## 4. 关键遗漏（除上述 3 个外）
- 无其他系统性遗漏。beta 模块深读不足已在 C-04/NEEDS_HUMAN_REVIEW 明示。

## 5. 错误升维检查
- KO-09 已确认需降级/收紧（单项目证据）：原标 L4 无 pending 标注 → 补 `Cross-project validation pending`
- 无 Pattern→Principle 的无据升维；无单案例→Pattern 违规（KO-01~06 均有 ≥2 EK + 对照项目）

## 6. 事实错误检查
- 无。抽查 10 条 Fact 全部在代码/文档中找到原文

## 7. Flow 错误检查
- 无。Control/State/Data/Evidence/Authority/Memory/Policy 七类流的关键 Edge 均通过 symbol 级回溯复核
- Authority Flow 补充：SecurityWatchdog 事件 handler 返回 dict（阻断信息）→ EventBus 阻止后续动作——已确认（security_watchdog.py:38-48）

## 8. 新 Benchmark / Regression Case（供 knowledge-archaeology-skill CI）
- **BK-01（Evidence Coverage）**：对含"安全/授权机制"的项目，考古必须显式检查 **bypass 路径**（IP 编码/重定向/新标签/glob 通配符）——browser-use 暴露了"域白名单考古若只查 allowlist 不查绕过防护 = Coverage MISSING"的教训。回归判断：安全类 EK 无 bypass 分析 → FAIL
- **BK-02（Flow Authority Edge）**：Authority Flow 必须包含"事件层阻断"Edge 的 symbol 级回溯（handler 返回 dict → EventBus 阻止）——防止把 Authority 写成"配置字段列表"
- **BK-03（Loop nudge 防误报）**：Agent 循环检测类 EK 必须覆盖"防误报设计"（escalating 分级/进展豁免），防止只写"检测到循环就提示"的简化

## 9. Reconciliation 建议（供阶段 6）
1. EK-08 补充 `block_ip_addresses`（IP 编码规范化阻断，profile.py:636 + security_watchdog.py:143-230）
2. EK-03 补充 escalating 分级（≥5/≥8/≥12 + 进展豁免）
3. EK-10 补充 vendor/outcome 三态 + step 计时重置
4. KO-09 标注 `Cross-project validation pending`
5. C-01 表述收紧为 Hypothesis
