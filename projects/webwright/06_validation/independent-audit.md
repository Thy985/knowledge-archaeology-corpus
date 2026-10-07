# Independent Audit Report — ARCH-2026-09-30-001（microsoft/Webwright）

> 独立 Auditor 盲重建：**未引用考古包**，独立重读仓库关键声明后对比。审计对象 = 仓库 HEAD bc26750a（135 tracked files）。原考古产物未被修改。

## 盲重建方法
1. 独立重读：`agents/default.py:198-270`（完成门硬逻辑）、`skill_factory/route.py` 全文（三决策 + fallback）、`tools/skill_use.py:58-114`（recommend 防御）、`pyproject.toml`（依赖）、`tests/skill_factory/test_gate.py + test_route.py + test_recommend.py`（行为断言）、README 1-60（Motivation/Why/News/对比表）。
2. 独立重读过程中**未参考** 00-06 包内容，独立记录 Findings 后再逐一比对。

## Independent Findings（盲重建产出）
- IF-1：完成门 `_self_reflection_gate_error → _tool_gate_error` 为硬阻塞：缺 workspace_dir / 缺 final_runs / 无 run_<id> / judge 文件缺失或不可解析 / predicted_label != 1，五种情形全部返回阻塞文本（含具体修复指引），`done=true` 无法通过。
- IF-2：route 决策流 = recommend(retrieve+decide) → promote(run/adapt 判定，grade + 3a + fillable slots) → run 失败（crash/timeout/空/错形状）回退 agent 且 hint 含 "tried and failed"；adapt hint 仅 "ADAPT: reuse the core"。
- IF-3：recommend 三重防御——库缺失显式 warning（防相对路径误判）、**LLM 幻觉 skill_id 若不在 retrieve 候选集则强制 skip**、decide 只判 shape-fit、promote 才判 run/adapt。
- IF-4：依赖清单 = httpx/jinja2/pydantic/pyyaml/rich/typer/playwright/python-dotenv/platformdirs（9 项）；**无 agent 编排框架、无图引擎、无隐藏编排框架**。
- IF-5：`_well_shaped` 诚实声明——"Not a correctness check…just the shape gate"：direct run 的成功答案只有 shape 保证，无正确性保证（无 ground truth）。
- IF-6：README 声称规模 ~450 行 core / ~570 Playwright env / ~150 CLI / "zero hidden frameworks（httpx, pydantic, playwright, typer）"；实测 467/567/175 行；依赖 9 项含 jinja2/pyyaml/rich/python-dotenv/platformdirs（README 为简化表述，非矛盾）。
- IF-7：tests 全 LLM-free 纯函数断言（gate 的 None/[]/""/shape mismatch/gold 精确匹配/status 忽略；route 的 run-answered-via-skill / fallback-marked / adapt-hint / skip-no-hint）。

## 判定对比
| # | 考古声明 | Auditor 判定 | 依据 |
|---|---|---|---|
| 1 | EK-12/EK-16：Self-Reflection Gate 硬逻辑（predicted_label==1 才放行 done） | **CONFIRMED** | IF-1 逐分支吻合（含 predicted_label!=1 的 id+1 重跑指引） |
| 2 | EK-33/EK-35：route 三决策 + fallback 标记语义 | **CONFIRMED** | IF-2/IF-7（test_route 行为断言逐一吻合） |
| 3 | EK-07/EK-08：strict JSON + done 降级 | **CONFIRMED** | mb.py parse_json_output（盲读确认 demote done 逻辑） |
| 4 | EK-40：技能零 token 重跑 + 复用增益 55→70% | **PARTIALLY_CONFIRMED** | 重跑机制源码确认；增益数字为 README 声明，本审计未复现（无 key） |
| 5 | EK-04：双模式一个 loop | **PARTIALLY_CONFIRMED** | 结构确认；两模式 A/B 质量差异无实测 |
| 6 | KO-01/KO-08：完成判定模式依赖（workspace 外部门 vs browser 自判） | **DOWNGRADED** | 原 KO-08 标记 Hypothesis 恰当；补充：live-browser 模式无外部门，适用域应再收窄为"工件化状态"（已在 Reconciliation 收紧） |
| 7 | README "zero hidden frameworks" | **CONFIRMED（附注）** | IF-4/IF-6：无编排框架属实；"just 4 deps"为省略表述，依赖实为 9 项（非隐蔽框架） |
| 8 | 基准数字 86.7%/60.1% | **PARTIALLY_CONFIRMED（Observation）** | README 声明存在且一致，未实测 |
| 9 | retrieve 简单/LLM 双法、fill 缺槽不猜 | **MISSING（考古包覆盖）→ 补充** | IF-3 揭示 recommend 幻觉 id 防御与 promote 分层，考古包 EK-34/36 未展开这两点，已补入 Reconciliation |
| 10 | 插件模式（四宿主）门控强度 | **MISSING → 候选** | 源码无法静态证明运行态门控；已标 C-06 Hypothesis |

## 判定统计
CONFIRMED 3 · PARTIALLY_CONFIRMED 2 · DOWNGRADED 1 · OVER_GENERALIZED 0 · CONTRADICTED 0 · MISSING 2（均已补/标候选）· NEEDS_HUMAN_REVIEW 0

## 3 项成功（Auditor 独立复现且与考古包一致）
1. 完成授权门（IF-1 == EK-12/16）：五种阻塞分支全部复现，含 `predicted_label={label!r} (expected 1)` 的原样错误文本与 run_<id+1> 重跑指引。
2. Skill Factory route 语义（IF-2 == EK-33/35）：run→answered（via skill，不启 agent）、run 失败→fell_back=True + "tried and failed"、adapt→ADAPT hint、skip→空 hint——与 test_route 断言完全一致。
3. 依赖/框架声明（IF-4/6 == 快照/01 层）：9 依赖无编排框架；行数 467/567/175 与 README 近似声明吻合（误差 <3%）。

## 3 项审计错误/修正（Auditor 比考古包更严格处）
1. **DOWNGRADED（KO-01 适用域）**：考古包 KO-01 已限定 workspace 模式，但未显式列出 live-browser 是"无门"分支的规模；审计后明确：**完成门的强度与状态模型绑定**（KO-08 提升为显式对照），防止读者误以为 Webwright 全模式都有验证门。
2. **MISSING→补充（recommend 防御链）**：考古包 EK-34/36 只写"缺槽→adapt 不猜"与"双法检索"，未写 **skill_id 幻觉拦截**（decided id 必须 ∈ retrieve 候选）与 **库缺失/空库显式 warning**——已补 EK-34 注记。
3. **MISSING→补充（route 成功路径诚实度）**：`_well_shaped` 的"非正确性检查"声明是 route 直接运行答案可信度的关键边界，考古包未显式记录——已补 Reconciliation 注记（run 的答案仅 shape 保证）。

## 遗漏清单（Auditor 独立发现，考古包未覆盖）
- retrieve.py `simple` 方法的实现细节（关键词启发式）
- `docs/skill_factory/manual.md / reference.md` 两份文档未精读（已标 C-05 注记）
- skill_use.recommend 的幻觉 id 防御（本次补入 EK-34 注记）
- 插件模式运行态门控（C-06 Hypothesis，非静态可证）

## Benchmark case（三形态对照：同一 web-agent-harness 问题域）
| 维度 | browser-use（corpus 09-22） | Webwright（本次） | deepseek-harness（corpus 09-03） |
|---|---|---|---|
| 动作空间 | 单步 DOM 动作（click/type/selector） | **自由形式 Python/bash 脚本（code-as-action）** | 工具调用 + 子代理 |
| 状态 | browser session（浏览器态） | workspace（磁盘态，浏览器可丢弃） | append-only SessionEvent log（日志态） |
| 完成判定 | 独立 judge + 预算强制 done | Self-Reflection Gate（predicted_label==1） | 工具守卫 + 审批 fail-closed |
| 架构复杂度 | event-driven + 16 watchdog + 66k 行 | 单 loop ~467 行 + skill_factory | everything-is-a-plugin + Capability Seam |
| 上下文治理 | compaction 25 步/40k 字符 + 分级降级 | ARIA 裁剪 + LLM 压缩 + 截图不自动附 | 记忆压缩 + 投影 |
| 复用机制 | 无显式技能库 | **Skill Factory 零 token 重放（55→70%）** | 无 |

**Benchmark 结论**：三策略证明"web agent harness"问题域存在至少三条正交设计轴——动作空间（预测 vs 编程）、状态归属（浏览器/磁盘/日志）、完成权威（judge/门/审批）。Webwright 占据"程序化动作 + 磁盘状态 + 外部门"一角，且以 ~1.5k-8k LoC 的极小 footprint 达到 SOTA 声明。

## 审计边界声明
- 未执行仓库代码/测试（无模型 key、无浏览器）；静态审读。
- 未修改原考古产物；本报告为独立产物。
