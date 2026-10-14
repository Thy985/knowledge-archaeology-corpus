# RAMPART — Archaeology Index

> 一句话定位：pytest-native 的 Agentic AI 安全测试框架（Microsoft AI Red Team），把"红队技术"变成可嵌入 CI 的安全测试。核心命题：**"安全测试框架如何让'不可观测'与'不安全'不被混淆，并让基础设施故障不被误报为安全发现"**。

- **repository**: https://github.com/microsoft/RAMPART.git（MIT, Python >=3.11, pyrit==0.13.0 pinned）
- **corpus 状态**: 活跃维护（两次考古：2026-09-04 initial + 2026-10-15 refresh）

## 考古 Run 列表
| Run | 日期 | Commit | Mode | 摘要 |
|---|---|---|---|---|
| [ARCH-2026-09-04-001](archaeology-runs/ARCH-2026-09-04-001/) | 2026-09-04 | `125595cb` | initial | EK 28 / KO 8 / 独立验证 CONFIRMED 18 + 实测脚本（三值逻辑真值表 / PayloadStore 逃逸 7/7） |
| [ARCH-2026-10-15-001](archaeology-runs/ARCH-2026-10-15-001/) | 2026-10-15 | `2c9067df` | refresh | EK 35 / KO 8 / 独立验证 CONFIRMED 21 + DOWNGRADED 1（trace.py 未接线）+ MISSING 2；观测即契约 / 判定与故障分离 / 信任边界贯穿三主线 |

## 产物结构（每次 run 完整独立）
```
archaeology-runs/<run_id>/
├── job_manifest.md / snapshot_artifact.md / 00_overview.md
├── 01_project-layer.md / 02_engineering-knowledge.md / 03_knowledge-layer.md
├── 04_flow-atlas.md / 05_candidates.md / 06_validation.md
├── independent_validation_report.md / run_metadata.yaml
```
> 顶层 00-06 与 metadata.yaml 为 2026-09-04 首轮产物；10-15 refresh 完整产物在 archaeology-runs/ARCH-2026-10-15-001/（不覆盖历史）。

## 关键认知（跨 run 稳定）
1. **观测即契约**：Adapter 声明观测通道，评估器按通道取信；缺通道→UNDETERMINED 非假 NOT_DETECTED；SAFE+RESPONSE_ONLY+零工具→强制降级 UNDETERMINED
2. **判定与故障分离**：基础设施异常→ERROR（从攻击成功率分母剔除，"503 不是安全发现"）；不完整运行强制非零退出（"green 必须意味着 ran and passed"）
3. **信任边界贯穿**：被测 agent 输出在 xdist 传输/终端渲染/payload 存储/LLM judge prompt 四处独立消毒
4. **（10-15 新发现）** trace.py 的 run_trace_async/evaluate_final_trace_async 为公共导出 helper（单测覆盖），当前内置策略（XPIA/SingleTurn）各自内联驱动循环——"先导出核心 helper 并测试、策略后续接入"的演进模式
