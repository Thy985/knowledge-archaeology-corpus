# 00 · Overview · ReviewBench（ARCH-2026-10-14-001）

## 项目一句话定位
ReviewBench 是 GitHub 官方开发的开源 AI 代码审查评测基准：以人工多源审校的 golden findings 为 ground truth，把任意候选 review agent 的 findings 经 **LLM 语义匹配 → 未匹配项 LLM 分类 → 评分** 三步管道，产出 grounded/augmented precision-recall 指标，用于在公开数据集上可复现地比较不同代码审查 Agent 的质量。

## 考古元信息
- **run_id**: ARCH-2026-10-14-001
- **repository**: https://github.com/review-bench/ReviewBench.git
- **commit**: `c17b05be9c90b85a1fc012ce39e455e2d09a7255`（main，2026-10-13，"Onboard block/review (block-review) (#71)"）
- **skill_version**: knowledge-archaeology v3.2
- **规模**: TypeScript 5758 行 / 33 ts；303 文件；v1.0.0；MIT
- **数据集**: corpus/manifest.json 219 PR（test set 25）+ golden/ 219 份金标

## 为什么值得考古（本次 Selection 理由摘要）
1. 评测可信度方法论三环之第三环（10-12 AgentJudgeBench=judge 审计层、10-13 Transect=结果可观测层、本次 ReviewBench=基准构造本身可信）；
2. Agent-AI Engineering 直接场景：代码审查 Agent 是当前 AI 工程最活跃场景之一；
3. 仓库是"不可信输入 → LLM 判定"的完整防御范式样本（提示词注入防护、git 安全层、egress 治理、契约校验），对任何评测/沙箱系统有直接可迁移价值；
4. 官方方法论自带 Threats to Validity（§8），是研究评测可信度的第一手反例素材。

## 本次考古核心发现（预览）
| 发现 | 层级 | 证据 |
|---|---|---|
| Findings 文本被当作"数据"而非"指令"处理（escape + data-not-instructions），防评测对象注入 judge | EK / L2 | scripts/eval/prompt-format.ts L7-14, 17-25 |
| 不可信 git 输入全链路 argv + hex-ref/nwo 校验 + 路径穿越拒绝 | EK / L2 | scripts/extraction/git.ts L32-72 |
| 匹配→评分分离：coveredGolden 全算 recall，scored match 只取第一个未认领 golden（防止"一鱼多吃"） | EK / L2 | scripts/lib/match-types.ts L30-68 |
| 基准自承四大威胁：golden 不完整、classifier 依赖、离线/在线收敛、matcher 依赖 | KO 反例 / L3-L4 | docs/METHODOLOGY.md §8 |
| 评测运行是"全有或全无"：单 PR 失败即整 run 失败；镜像 digest-pinned 保证字节级可复现 | EK / L2 | AGENT_CONTRACT.md L91-92；schema/agent-manifest.schema.json image 字段 |

## 认知状态声明（Epistemic）
- 本包所有 L1/L2 事实均可回溯到仓库实际文件与行号；
- L3+ 泛化结论标注 `ReviewBench-verified · Cross-project validation pending`，不冒充跨项目已验证原则；
- 凡证据不足的内容进 05 Candidates，不升层。

## 产物清单
| 文件 | 内容 |
|---|---|
| 01_project-layer.md | 项目地图（是什么/怎么运行/模块/配置/契约/治理） |
| 02_engineering-knowledge.md | EK Graph（24 条 EK + 六类边） |
| 03_knowledge-layer.md | 8 个 KO（L3-L5 + aggregation_rule R1-R4） |
| 04_flow-atlas.md | 七类流（Control/State/Data/Evidence/Authority/Memory/Policy） |
| 05_candidates.md | 未验证假设与跨项目候选 |
| 06_validation.md | Truth/Coverage/Flow/Abstraction/Counterexample/Epistemic + Blind Reconstruction + 反例 |
| run_metadata.yaml | 质量指标与验证结果 |

## 三层配比（交付前核对）
| 层 | 本次产出 | 目标参考 |
|---|---|---|
| Project Layer | 1 地图（完整） | 1 |
| Engineering Knowledge | 24 条（全带 links） | 40-60（小中型项目按比例缩放，不硬凑） |
| Generalized KO | 8 个（全带 aggregation_rule） | 7-12 |
| Candidates | 5 条 | 未验证假设 |
