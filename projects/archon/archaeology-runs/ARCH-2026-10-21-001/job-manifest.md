# Job Manifest — ARCH-2026-10-21-001

| 字段 | 值 |
|---|---|
| job_id | ARCH-2026-10-21-001 |
| project | Archon |
| repository | https://github.com/coleam00/Archon.git |
| mode | initial |
| priority | P0 |
| source_entry | 03_expansion_queue/candidates/[cand]agent-containment-breach-2026.md（Day3 并入：CVE-2026-111322 第 4 类证据线）+ 07_radar_watch/current_focus.md（Harness 周 Day3） |

## 候选排序（2026-10-21，Harness/治理 周 Day3）

| 排序 | 项目 | 状态 | 评级 | 核心理由 |
|---|---|---|---|---|
| **1** | **Archon**（coleam00/Archon） | NEW | **P0** | Harness 本体（"first open-source harness builder for AI coding"）；Day3 雷达 CVE-2026-111322（控制面默认无认证→RCE，CVSS 9.8）＝本周最重要治理反例；23,733★ pushed 10-20 考古当日仍活跃；与 box/mxc/governance-toolkit 形成"治理成败对照对"；周五事件集群精读直接受益 |
| 2 | agent-aegis-governance | QUEUED | B | guardrail 策略引擎对比池（Aigis/MS AGT 同列），周五对比期再入池，非单项目最高优先 |
| 3 | agent containment breach 集群 | QUEUED | A | 威胁情报/事件分析（非单项目仓库，无法浅克隆六阶段考古）；周五精读回填 E 层 |
| 4 | Strands Box | COMPLETED | — | 10-20 已考古（ARCH-2026-10-20-001，PR #49） |
| 5 | MXC | COMPLETED | — | 10-19 已考古（ARCH-2026-10-19-001，PR #48）作对照基准 |

## 选中理由（7 条）
1. **领域中心度**：本周即 Harness/治理 周；Archon 自称 "the first open-source harness builder for AI coding"——是对领域研究对象本体的直接考古（A 层 Harness 架构），高于 B 层遏制/防火墙品类
2. **Novelty + 事件驱动**：Day3 雷达刚并入 CVE-2026-111322（coleam00/Archon 0.2.13–0.11.1 `/api/*` control API 无认证且默认绑 0.0.0.0 → 未授权 RCE，CVSS 9.8）；"secure-by-default" 反例是最高价值学习素材
3. **增长信号**：23,733 stars；pushed_at 2026-10-20（考古当日仍活跃）；CVE 后版本迭代 v0.12.x（修复版）
4. **未考古**：corpus 34 项目无 archon（NEW，无 refresh 冲突）
5. **Corpus 连接**：与 Strands Box（10-20，OS 遏制防御侧答案）/ MXC（10-19，OS 隔离）/ agent-governance-toolkit（无状态策略引擎）/ openclaw（已考古 agent 框架）组成**治理成败对照对**——同一周内既有防御侧正向案例又有配置反例，可升维"控制面安全"认知模型
6. **Benchmark 学习价值**：agent harness 架构（deterministic/repeatable）、控制面治理、MCP 信任边界、默认安全缺失的完整失败路径——Gold Record 级案例
7. **周五计划支撑**：事件集群精读回填 E 层（含 Archon RCE 链条）——考古 Archon 本体为回填提供源码级证据

## 排除项
- Strands Box / MXC：COMPLETED（不重复考古；各自作为对照基准）
- agent containment breach 集群：威胁情报非单项目仓库，无法执行浅克隆+六阶段考古
- agent-aegis：B 级同品类，周五 guardrail 对比池扩容时再评估
