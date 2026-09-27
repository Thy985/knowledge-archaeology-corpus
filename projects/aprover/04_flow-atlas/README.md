# Flow Atlas — AProver（七类流）

> 全部边可回溯到 symbol/file/condition/state transition（详见 06 追溯表）。
> commit b9314c7 ｜ run ARCH-2026-09-28-001

## 1. Control Flow（控制流）
```
CLI(bmc_agent.cli:main) → AMCPipeline.run(source_file, driver_name)
  → Pass1 tree-sitter 解析 + 两遍全局调用图构建 [C]
  → Pass1.5 DomainAnalyzer 领域摘要 [L]
  → Phase1 SpecGenerator（SCC 分层拓扑序生成 spec; dual-spec 分歧标记）[L]
  → Phase1.5 FlagSelector（per-function 旗标, opt）[L]
  → Phase2 BMC Engine（harness 合成 + CBMC/Kani/JBMC 求解, k=4/120s）[C]
  → _dedup_counterexamples（per (fn, property_type)）[C]
  → Phase3 CExValidator.validate（S1 reachability → S2 feasibility → S3 dynamic(opt) → S4 realism(opt)）
     ├ REAL_BUG → BugReporter.create_report（五档分级）
     ├ SPURIOUS → Refiner 提议紧前置条件 → Soundness Guard（BMC over-refine 检查）
     │           → accepted → Phase3c 重跑该函数 → Phase3b 重跑调用者 → 回 Phase3（capped re-queue）
     │           → rejected → UNRESOLVED
     └ UNRESOLVED → track + skip（never silently dropped）
  → Phase4 SpecQuality（opt, BMC_AGENT_ENABLE_SPEC_QUALITY）
  → artifacts/ 落盘（spec.json / cbmc_result.json / bug_report.json / ...）
```
关键条件/状态转移：
- `_prop_is_reach()` 仅在 SVCOMP_PROP=unreach 时为真（属性类门控，非 benchmark 门控）
- 属性类免疫：`(fn_name, _prop_type(failing_property)) in feedback_cleared` → 跳过同属性类兄弟 CEx
- `_cited_caller_is_fabricated(cited, func_source_file)` 防 LLM 编造调用者
## 2. State Flow（状态流）

函数级验证状态机（每函数）:
  pending → spec_d (SpecGenerator 产出 Spec) → bmc_d (BMC 产出 CEx/clean)
  → classified (CExValidator 三态) → refined (Soundness Guard 通过后精化 spec)
  → [clean | bug(五档) | unresolved]

关键状态与转移:
- UNRESOLVED 状态永不被静默删除（cex_validator docstring: never silently dropped）
- realism 降档: REAL_BUG + UNREALISTIC → unlikely（re-tier, 非 delete）
- confirmed_dynamic 免疫 realism 降档（main.assertion.N 例外）
- 属性类免疫: feedback_cleared 集 (fn, prop_class) 一经 VERIFIED CLEAN，同属性类兄弟 CEx 全跳过
- Spec.disagreement 标记（dual-spec 分歧, best-effort）
- CExDedup: per (function, property_type) 保留一个, assertion.N 全保留

## 3. Data Flow（数据流）

- 源码 → tree-sitter CST → global call graph（两遍构建）
- Spec(dataclass: pre/post/category/JSON reasoning/DSL 谓词) → dsl_to_cbmc.translate_atom → __CPROVER_assume/assert C 语句 + harness
- DS 谓词翻译: valid_string→ptr!=NULL; valid_range→ptr!=NULL && lo>=0 && hi>=lo; in_bounds→idx 范围; 自然语言→注释
- Counterexample(failing_property/trace) → ValidationResult(outcome + reachability/feasibility error 标志 + is_latent_bug)
- SPURIOUS → refined precondition（Refiner）→ 回 BMC 验证
- REAL_BUG → RealismVerdict → BugReport(confidence 五档 + reasoning trail)
- spec_evidence: CallerEvidence(callers/address_taken_sites/doc clauses) → v2 spec 生成输入

## 4. Evidence Flow（证据流）

- 证据来源分层: [C]确定性(solver/runtime) > [L]agentic(LLM 判定)
- spec_evidence.harvest_callers: 调用点/地址取用点证据束 → spec 生成（过滤测试路径/字符串/注释）
- cex_validator: S1 reachability(BMC 子查询 + LLM fallback) / S2 feasibility(真 callee vs stub) / S3 dynamic(signal 捕获)
- realism: witness-pattern 确定性短路(UNREALISTIC high-conf) vs LLM 审计(realistic/unrealistic/uncertain)
- 证据分级五档: confirmed_dynamic > confirmed_system_entry > confirmed_bmc > likely > unlikely
- 证据保留: UNRESOLVED track+skip; realism 降档保留审计轨迹; UNREALISTIC 不被丢弃

## 5. Authority Flow（权威流）

- soundness_policy.py = 删除/降级判定的单一权威源（justification→action 映射）
- 权威授予: DETERMINISTIC_VERIFIER→DELETE; SELF_VERIFYING_WITNESS→DELETE; AGENTIC_JUDGMENT→RETIER 仅
- refiner 授权条件: CBMC 排除 AND 调用点确定性检查 → DELETE; 否则 RETIER
- witness 授权条件: 精确复现同一故障（真实公共 API + 匹配 CBMC 属性）→ DELETE
- realism 只可降档（unlikely），不可删除；confirmed_dynamic 免疫（main.assertion.N 例外）
- 每函数隔离: callee postcondition 以 __CPROVER_assume 编码为 stub 契约（权威由 spec 赋予）

## 6. Memory Flow（记忆流）

- artifacts/ 落盘: spec.json / refinement_history.json / cbmc_result.json / classification.json / bug_report.json / harness / propagation_events.json
- findings/ 研究档案: VibeOS(13 bugs/675 fn) / llm_c(22/30 clean) / linux_drivers(8 findings) / libarchive / nghttp2 / oss-curl / llama_cpp_ggml / aws_neuron_driver
- 会话记忆: SESSION_SUMMARY_2026-05-21..22 / JUDGMENT_NOTES.md / 各 sweep 结果 json
- 反馈记忆: feedback_loop / spec_quality(变异/覆盖/一致性) → 自改进闭环
- realism 模式库记忆: witness-pattern 以事件驱动演进(2026-05-13 jq / 05-18 ch341+pl2303 / 06-12 vacuity)

## 7. Policy Flow（策略流）

治理闭环 Decision → Approval → Policy → Enforcement → Future Decision:
- Decision: 项目设计原则 = agents propose, conventional tools dispose（README）
- Approval: soundness_policy.py 单一事实源（模块 docstring 即 ADR: 删除不对称性论证）
- Policy: PIPELINE.md(实现 ASCII 自述) + README 配置表 + PLAN_autonomous_mode.md(自主模式)
- Enforcement: resolve_action/refiner_exclusion_action/witness_action 在所有可能 drop finding 的决策点被调用
- Future Decision: realism witness-pattern 库随失败演进; Phase4 spec_quality 反馈; tests 漂移(EK-16)提示需同步

---
## Flow→KO 交叉校验
- KO-01(删除不对称) ← Authority Flow 全线 + State Flow UNRESOLVED 保留
- KO-02(信任链) ← Control Flow 管线序 + Data Flow 翻译链 + Evidence Flow 分级
- KO-03(确定性短路) ← Evidence Flow witness-pattern + State Flow 属性类免疫
- KO-06(智能≠权威) ← Authority Flow justification 映射
全部 KO 与 Flow 无矛盾（06 交叉校验表）。

