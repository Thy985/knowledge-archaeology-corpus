# 06 Validation / Reconciliation（ARCH-2026-11-16-001）

## 1. 输入
- `06_validation/self-validation.md`：生产侧自检，**PASS WITH LIMITATIONS**。
- `06_validation/independent-validation.md`：独立 Auditor，40 条 IF，判定 **26 CONFIRMED / 5 PARTIALLY_CONFIRMED / 0 DOWNGRADED / 0 OVER_GENERALIZED / 4 CONTRADICTED / 5 MISSING**。
- 目标仓库：`https://github.com/OWASP/www-project-agent-memory-guard.git` @ `b63f76eece80d84387b6056919bd94816c60bd89`。

## 2. 逐项裁决
| 独立项 | 判定 | 裁决 | 落点 |
|---|---|---|---|
| IF-36 `boundary` 伪边 7 处 | CONTRADICTED | **接受**：`boundary` 不在六类边 schema；改为 `contrast`/`constraint`，重算 166 edges | `02_engineering_knowledge.md` |
| IF-37 EK 48 vs 55 | CONTRADICTED | **接受**：以脚本计数 55 为准 | `00_overview.md`、`02` 文末 |
| IF-38 C-14 vs 15 | CONTRADICTED | **接受**：C-01…C-15；独立新增后为 C-19 | `00`、`05` |
| IF-39 `events()` vs property | CONTRADICTED | **接受**：改为 `events` property | EK-14/38/55 |
| IF-27/EK-27 安全修复版本归属 | CONFIRMED（包内初版过窄） | **接受**：改为“release 之后、当前 tag 含、CHANGELOG 仍列 Unreleased” | EK-27、C-15、HR-05 |
| IF-33 KO-03 | PARTIALLY_CONFIRMED | **保留 L3，不升 L4**：事件存在但无消费者/无上限；已在反例中写明 | `03` KO-03 |
| IF-34 KO-04 | PARTIALLY_CONFIRMED | **保留 L3，L4 candidate 不升级**：图与门禁存在，但 `verified_by` 可伪造、target 归一化不一致 | `03` KO-04 |
| IF-35 KO-05 | PARTIALLY_CONFIRMED | **保留 L3，标 Cross-project pending**：Llama 旁路强，但 LangChain 读路径受控；不能写成“所有适配器都绕过” | `03` KO-05、`05` XP-02 |
| IF-40 KO-07 L4 | PARTIALLY_CONFIRMED | **不降级**：包内已标 “L4 候选”，不是已验证 L4；保留 capability 分离作为候选 | `03` KO-07 |
| M-01 set_messages 索引错位 | MISSING | **接受为 C-16（Hypothesis）**，不改 KO/EK | `05_candidates.md` |
| M-02 retire predicate 异常吞掉 | MISSING | **接受为 C-17（Fact/Hypothesis）** | `05_candidates.md` |
| M-03 `/read` found=None 语义 | MISSING | **接受为 C-19（Hypothesis）** | `05_candidates.md` |
| M-04 全拦时旧消息残留 | MISSING | **接受为 C-18（Hypothesis）** | `05_candidates.md` |
| M-05 旧 KO 差异表缺失 | MISSING | **接受为 HR-06 + BC-01 FAIL**；不伪造旧 KO 状态 | `05_candidates.md`、`06_validation/independent-validation.md` |

## 3. 不接受的项
无。独立审计没有提出与源码证据相冲突、需要驳回的断言；4 条 CONTRADICTED 全部是包内初版错误，不是仓库事实争议。

## 4. 设计意图 / 实现 / 测试 / 运行时分层
| 主题 | 设计意图（文档） | 实现 | 测试 | 运行时 |
|---|---|---|---|---|
| “每条消息都过 guard” | `GuardedChatStore` docstring | 只对写入；`get_messages` 直读 | denied-delete 测试默认接受读旁路 | fake backend 复现 outsider 读 `SECRET` |
| 10 detector / 4 类分类 | `design.md` | 默认 7 / `MemoryClass` 6 类 | 默认套件测试 | 探针 `len=7` |
| read 先查 quarantine | `design.md` | `_read_impl` 不查 `_quarantine` | 未见对应测试 | 未跑真实 server |
| 0.4.0 发布含 redaction 修复 | CHANGELOG `[Unreleased]` / PyPI | 当前 tag `v0.4.0` 含修复；release commit 早于修复 | 504 tests 通过 | 未核验 PyPI 构建 |

**裁决**：以上均作为 `CONDITIONAL` 记录，不把文档当实现事实；`00`/`02` 已按此表述。

## 5. 知识层影响
- **不新增 KO**：独立发现均为边界/实现缺口，未形成跨项目 Pattern；C-16…C-19 留在 Candidates。
- **不删除 EK**：55 条 EK 全部保留；仅修 schema（7 处边类型）与计数。
- **不升 L4/L5**：KO-03/04/05 保持 L3；KO-07 保持 L4 candidate；无 L5。
- **反例保留**：每个 KO 的反例段落原样保留；独立新增反例写入 05，不塞进 03。

## 6. 质量指标（最终）
| 指标 | 值 | 口径 |
|---|---|---|
| EK | 55 | 脚本计数 |
| EK edges | 166 | 六类边逐段匹配 |
| avg edges/EK | 3.02 | 166/55 |
| 非法边 | 0 | 无 `boundary` |
| 游离 EK | 0 | 每条 ≥2 边 |
| KO | 10 | 每个有 R1–R4 |
| Flow edges | 49 | 七类计数 |
| Candidates | 19（C-01…C-19） | 含独立新增 4 条 |
| XP / HR | 5 / 6 | XP-01…05；HR-01…06 |
| 测试 | 504 passed / 2 skipped / 10 xfailed | Python 3.12, pytest 9.0.2 |
| Independent | 26 C / 5 P / 0 D / 0 OG / 4 CT / 5 M | 40 IF |
| Benchmark | BC-01 FAIL（coverage） | 旧 KO 差异表缺失 |

## 7. Skill Evolution 判断
- **是否触发 Candidate Skill PR**：**否**。
- **原因**：本轮暴露的是 refresh 模式产物缺口（BC-01/M-05），属于 P2 流程改进，不是系统性错误；不满足“High/Medium 风险必须 Owner 审查”之外的自动合并条件，也不值得为本轮单独开分支。
- **记录**：HR-06 提议在 skill 的 refresh 分支强制 `refresh-delta.md`（字段：`old_ko_id / old_claim / new_status / superseded_by / evidence_change`）。该提议只写入 corpus，不改生产 Skill。
- **未修改**：目标仓库、KnowlegeMap、生产 Skill v3.2。

## 8. Corpus 写入策略（不覆盖历史）
- 旧 run `projects/agent-memory-guard/archaeology-runs/ARCH-2026-10-07-001/` **原样保留**。
- 新 run 写入 `projects/agent-memory-guard/archaeology-runs/ARCH-2026-11-16-001/`，包含完整 00–06 + `run_metadata.yaml`。
- 顶层旧 `00_overview.md` / `01-project-layer` / … / `06-validation` **不覆盖**；只新增 `REFRESH-2026-11-16.md` 作为增量索引。
- 若 Corpus 已有同名 run 目录：停止并改用 `ARCH-2026-11-16-001-r2`，不覆盖。
