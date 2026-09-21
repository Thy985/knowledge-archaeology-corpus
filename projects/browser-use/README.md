# browser-use — Knowledge Archaeology

**一句话定位**：开源事实标准的浏览器 Agent 库（115,743★，v0.13.10）——LLM 决策 + CDP 控制的网页自动化 Agent，同时是三层商业漏斗的 OSS 顶端。

## 产物清单
| 目录/文件 | 内容 |
|---|---|
| `metadata.yaml` | 考古元数据（run_id / commit / skill_version） |
| `00_overview.md` | 概览：一句话定位 + 核心数字 + 知识贡献 |
| `01_project-layer/` | 项目地图（架构/模块/数据结构/配置/治理） |
| `02_engineering-knowledge/` | EK Graph（22 条 + 六类边） |
| `03_knowledge-layer/` | Generalized KO（10 个，R1-R4 聚合规则） |
| `04_flow-atlas/` | 七类流（Control/State/Data/Evidence/Authority/Memory/Policy） |
| `05_candidates/` | 未验证假说（5 个跨项目候选） |
| `06_validation/` | 验证报告 + 独立审计 + Reconciliation |
| `archaeology-runs/ARCH-2026-09-22-001/` | 本次 Run 原始产物（manifest/snapshot/metadata/审计） |

## 考古日期与版本
- 考古日期：2026-09-22（run ARCH-2026-09-22-001，mode: initial）
- 目标版本：browser-use 0.13.10 @ `d8110c5`
- Skill 版本：knowledge-archaeology v3.2
- 独立验证：8 CONFIRMED / 3 PARTIAL / 1 DOWNGRADED / 1 OVER_GENERALIZED / 3 MISSING / 1 NEEDS_HUMAN_REVIEW；0 CONTRADICTED

## 关键认知速览
1. **软门控 vs 硬门控**（KO-03/07/09）：循环检测只注入上下文 nudge 从不阻断动作；预算/失败用不可绕过的硬门控——"行为质量靠说服，资源终止靠强制"
2. **上下文预算分级降级**（KO-02）：compaction → 75% 预算警告 → 最后一步强制 done → 失败强制 done
3. **事件驱动 watchdog**（KO-04）：bubus EventBus + 16 监视器（安全/下载/captcha/权限/DOM/崩溃/HAR）
4. **三重安全边界**（KO-05）：域白名单三层检查 + 重定向捕获 + `block_ip_addresses` IP 编码绕过防护
5. **OSS 漏斗产品化第二实例**（C-02）：与 mem0 同构（OSS 库→云端→自有模型推荐）
