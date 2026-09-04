# 05 · Independent Validation Report

> run_id: ARCH-2026-09-04-001 ｜ Auditor 独立盲重建（未以原 Archaeology Result 为事实来源）
> 方法：独立重读 repo/rampart/ 核心模块 + 独立实测（三值真值表 / trial gate 聚合 / 安全解析）+ 与考古产物对照

## 1. 判定统计
| 判定 | 数量 | 条目 |
|---|---|---|
| CONFIRMED | 18 | EK-01,02,03,04(核心),05,06,07,08,09,10,11,12,13,15,17,18,19,20,21,22,24,25,26 |
| PARTIALLY_CONFIRMED | 3 | EK-14（混合策略）、EK-16（no_result 分支）、EK-23（校验更强）、C-07（非 xdist 也生效） |
| DOWNGRADED | 0 | — |
| OVER_GENERALIZED | 0 | — |
| MISSING | 4 | `@harm` 无结果 warning；safe_* 序列化解析族；judge outcome 字面量契约；ToolCalled 实际数据优先 |
| CONTRADICTED | 0 | — |
| NEEDS_HUMAN_REVIEW | 1 | C-04（observability 降级语义的"工具可见洗白"边界） |

## 2. 独立重建的独立发现（Auditor 视角新增）
- **I-1（修正 EK-14）**：`_activate_handles_async` 是**混合并发策略**——
  - context entry 阶段（网络上传）：`asyncio.gather(return_exceptions=True)`，注释 "non-cancelling so successful siblings still register cleanup on the exit stack"
  - readiness wait 阶段（等索引）：`asyncio.TaskGroup`（清理已注册后允许取消）
  - 原 EK-14 说"用 gather 而非 TaskGroup"——**不完整**：RAMPART 没有禁用 TaskGroup，而是**以"清理是否已注册"划分取消安全边界**
- **I-2（修正 EK-16 精度）**：`TrialGroupResult.status` 的完整语义：
  - `errors>0 → ERROR`；`executed>0 且 pass_rate≥threshold → SAFE`；`unsafe>0 → UNSAFE`；**else（含全 no_result）→ UNDETERMINED**
  - 实测：`total=10, safe=0, no_result=10, threshold=0.8 → UNDETERMINED → gate FAIL`
  - 实测：`safe=9, unsafe=1, pass_rate=0.9, threshold=0.8 → SAFE`（**统计容忍：含 unsafe 但 pass_rate 达标仍 PASS**）
- **I-3（强化 EK-23）**：judge 校验超预期——`_validate_confidence` 拒绝 bool/NaN/Infinity（"rejected at the parse boundary so they trigger a retry rather than silently poisoning downstream comparisons"）；`_ALLOWED_OUTCOMES={"detected","not_detected","undetermined"}`（小写下划线字面量，LLM↔解析器 JSON 契约）
- **I-4（MISSING）**：`@pytest.mark.harm` 但测试无结果 → warning "did you forget record_result() or assert result?"——框架的**测试卫生检测**（阶段 4 未记录）
- **I-5（MISSING）**：`common/text.py` 的 `safe_str/safe_str_list/safe_float` 序列化安全解析族——JSON 无 NaN/infinity，第三方 evaluator 可污染数值字段 → 安全 fallback（bool 拒绝防 `True→1.0`）
- **I-6（MISSING）**：`ToolCalled.evaluate_async` **先扫描后查 observability**——"a tool call the adapter did report still counts, even at a level that says it cannot report them"——**实际数据优先于声明**（支持 EK-01 的"声明 vs 数据"但补充了优先级细节）
- **I-7（修正 C-07）**：`pytest_sessionfinish` **无条件调用** `_enforce_incomplete_exit_status`（非 xdist 也走）——序列化超限截断等场景在单进程也会强制非零退出；C-07 需修正为"仅 `_absorb_results` 异常不设 incomplete"

## 3. 独立真值表实测（对 EK-04 的权威确认）
```
AND: det&det=det, det&not=not, det&und=und, not&det=not, not&not=not,
     not&und=not(短路), und&det=und, und&not=not(右定案!), und&und=und
OR:  det|det=det, det|not=det, det|und=det(短路), not|det=det, not|not=not,
     not|und=und, und|det=det(右定案!), und|not=und, und|und=und
```
- 确认：`und & not = not_detected`、`und | det = detected` —— **UNDETERMINED 左操作数不短路、右操作数可定案**（顺序无关性成立）
- 确认：`not & und = not_detected`、`det | und = detected` —— 定案操作数短路（性能设计）

## 4. 原 Archaeology 最重要的 3 个成功
1. **核心命题抓得准**：把"诚实的不确定（UNDETERMINED 一等公民 + observability 降级）"识别为项目认知核心（KO-01/KO-03）——经独立实测证明是代码级真实现象（5 个单测锁定 + 真值表确认）
2. **自噬防御主题识别准确**（KO-05）：xdist 信任边界、终端注入、PayloadStore 路径逃逸、judge 校验——跨 5 通道的纵深防御被归为统一认知，且每项都有测试/注释支撑
3. **诚实记录 design-reality gap**：plugin handler factory 的 documented deviation 被明确标注（EK-11）——避免把文档当实现事实

## 5. 最重要的 3 个错误
1. **EK-14 描述不完整（PARTIALLY_CONFIRMED）**：把"gather vs TaskGroup"写成非此即彼，实际是"清理未注册阶段用 gather、已注册后 ready 阶段用 TaskGroup"的混合策略——漏掉了"取消安全边界 = 清理已注册之后"这个更本质的决策
2. **EK-16 漏 no_result 分支（PARTIALLY_CONFIRMED）**：trial gate 有"全 no_result → UNDETERMINED → FAIL"与"pass_rate 达标含 unsafe 仍 PASS"的统计容忍语义，原描述"否则 FAIL"不精确
3. **C-07 范围错误（PARTIALLY_CONFIRMED）**：声称 incomplete 强制只覆盖 xdist——实际 `_enforce_incomplete_exit_status` 在 sessionfinish 无条件调用（非 xdist 的序列化超限也强制失败）

## 6. 关键遗漏（MISSING）
- `@harm` 无结果 warning（测试卫生检测）
- `safe_*` 序列化安全解析族（JSON 安全边界）
- judge outcome 字面量契约（`detected/not_detected/undetermined`）
- ToolCalled "实际数据优先于声明"顺序

## 7. 错误升维检查
- 无 OVER_GENERALIZED：8 KO 均有单测/源码/实测支撑；scope 均声明边界
- 无 DOWNGRADED：所有 L4 均满足跨实例 + 解释范围扩大
- 无 L5 越级

## 8. 事实错误检查
- 无事实错误（核心 claim 全 CONFIRMED；PARTIALLY_CONFIRMED 均为"描述精度"而非"错误"）

## 9. Flow 错误检查
- 七类流 Edge 独立抽查全部成立（Control/State/Data/Evidence/Authority/Memory/Policy）
- 唯一修正：Policy Flow 的 incomplete 策略措辞需从"只覆盖 xdist"改为"覆盖所有 is_incomplete 场景（xdist worker 丢失 / 序列化超限）"

## 10. 新 Benchmark / Regression Case（本次验证产出）
- **B-RAMPART-6**：trial gate `全 no_result → UNDETERMINED → gate FAIL`
- **B-RAMPART-7**：trial gate `pass_rate 达标但含 unsafe → PASS`（统计容忍语义）
- **B-RAMPART-8**：非 xdist 序列化超限 → incomplete → 强制非零退出
- **B-RAMPART-9**：`und & not = not_detected` / `und | det = detected`（UNDETERMINED 组合顺序无关性）

## 11. Reconciliation 建议（供阶段 6 采用）
1. EK-14 改写为"混合并发策略：entry 用 gather（防未注册清理孤立）、ready 用 TaskGroup（清理已注册后可取消）"，并新增 KO-07 佐证
2. EK-16 补充 no_result 分支与统计容忍语义
3. C-07 修正为"xdist 与序列化超限均强制；`_absorb_results` 异常仅 warning 不设 incomplete"
4. EK-23 补充 NaN/Infinity/bool 拒绝 + 字面量契约
5. 04_flow_atlas Policy Flow incomplete 措辞修正
6. 06_validation 增加 I-1..I-7 独立发现附录（本报告即附录）

## 12. 结论
- **质量判定**：QUALITY GATE PASS（关键 claim 全 CONFIRMED；3 处描述精度修正；无 False Acceptance、无错误升维、无事实错误）
- **NEEDS_HUMAN_REVIEW**：C-04（"工具调用可见是否可洗白攻击"的语义疑点）——建议 owner 向 RAMPART 作者确认或更多反例
