# RAMPART — Knowledge Archaeology Index

**Repository**: https://github.com/microsoft/RAMPART.git（pytest-native Agentic AI 安全/安全测试框架，MIT，Alpha）

| Run | Date | Mode | Commit | Summary |
|---|---|---|---|---|
| [ARCH-2026-09-04-001](archaeology-runs/ARCH-2026-09-04-001/) | 2026-09-04 | initial | `125595c` | 28 EK + 8 KO；XPIA/SingleTurn 执行策略、Evaluator 三值组合、xdist 信任边界、PayloadStore 防护 |
| [ARCH-2026-10-18-001](archaeology-runs/ARCH-2026-10-18-001/) | 2026-10-18 | **refresh** | `0fb9cc2` | +17 EK（R01~R17）+ 5 KO；trace runner / versioned serializer / compatibility gate / trial population / RAMPARTner skill |

## 产物导航（refresh ARCH-2026-10-18-001）
- [00 Overview](archaeology-runs/ARCH-2026-10-18-001/00-overview.md)
- [01 Project Layer](archaeology-runs/ARCH-2026-10-18-001/01-project-layer.md)
- [02 Engineering Knowledge（EK Graph）](archaeology-runs/ARCH-2026-10-18-001/02-engineering-knowledge.md)
- [03 Knowledge Layer（KO）](archaeology-runs/ARCH-2026-10-18-001/03-knowledge-layer.md)
- [04 Flow Atlas（七类流）](archaeology-runs/ARCH-2026-10-18-001/04-flow-atlas.md)
- [05 Candidates](archaeology-runs/ARCH-2026-10-18-001/05-candidates.md)
- [06 Validation & Evidence](archaeology-runs/ARCH-2026-10-18-001/06-validation-evidence.md)
- [07 Independent Validation](archaeology-runs/ARCH-2026-10-18-001/07-independent-validation-report.md)
- [Reconciliation](archaeology-runs/ARCH-2026-10-18-001/reconciliation.md)
- [Run Metadata](archaeology-runs/ARCH-2026-10-18-001/run_metadata.yaml)

## 认知主线（6 周演进）
> 09-04 → 10-18：RAMPART 把"评估"从隐式约定演进为显式契约——trace runner 统一执行、versioned serializer 持久化、compatibility gate 治理契约演化、trial population 取代克隆、RAMPARTner 固化 onboarding。核心认知：**终局证据与执行证据分离（KO-R01）/ 契约兼容性=显式决策（KO-R02）/ 判定者与被判定历史隔离（KO-R03）**。

*09-04 扁平产物（00-07 + metadata.yaml）保留为 initial 基线，未被 refresh 覆盖。*
