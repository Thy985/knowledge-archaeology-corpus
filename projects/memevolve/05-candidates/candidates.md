# Candidates — 未确认假说 / 待验证模式 / 跨项目候选

> 纪律：Candidates 不是 KO。不得在 KO/结论中引用为 Fact。每条标注：类型 / 当前证据 / 缺失证据 / 验证路径 / Epistemic。

## C-01 Lightweight 长期记忆的使用量索引在剪枝后会指向错误条目
- **类型**: 项目内缺陷假说
- **当前证据**: `used_memory_ids` 形如 `strategic_3`，更新时按下划线解析索引（`lightweight_memory_provider.py:501-556`）；长期剪枝会删除元素并重建列表（`:1159-1341`）。
- **缺失证据**: 未运行“写入→剪枝→再用”序列；未确认解析函数在越界时是否回退。
- **验证路径**: 单测构造 5 条长期记忆，标记 `strategic_3`，触发剪枝删除 `strategic_1`，再更新 `strategic_3`，断言分数落到正确 uuid。
- **Epistemic**: Hypothesis · **影响**: 若成立，Evolver/Lightweight 的成功率统计会失真。
- **关联**: EK-32、EK-31、E-049

## C-02 多线程 provider 实例写同一 JSON 时无锁，存在最后写入覆盖
- **类型**: 项目内并发假说
- **当前证据**: runner 注释“each thread creates independent instance”（`run_flash_searcher_mm_gaia.py:594-607`）；所有 provider 以整文件 `json.dump` 持久化（E-090）；无 `fcntl`/`filelock`。
- **缺失证据**: `run_provider` 固定 `concurrency=1`（E-078），主路径无并发写；只有外部直接调用 runner 且 concurrency>1 时才可能触发。
- **验证路径**: 以 `--concurrency 4`（若 runner 支持）跑同一 provider，比较写入条数与预期。
- **Epistemic**: Hypothesis · **影响**: 记忆丢失但无报错。
- **关联**: EK-50、E-080、E-090

## C-03 自动修复会把“本应失败”的候选修到通过，抬高通过率
- **类型**: 项目内治理假说
- **当前证据**: `enable_auto_fix=True` 默认（E-067）；修复后重跑快检（E-069）；通过则回写真实目录（E-068）。
- **缺失证据**: 未统计修复前后通过率；未做“关闭 auto-fix”的对照实验。
- **验证路径**: 对同一批 LLM 生成候选，分别在 auto-fix=True/False 下跑，记录 verdict 分布与最终锦标赛成绩。
- **Epistemic**: Hypothesis · **影响**: 质量门失去区分度；KO-07 的实证基础。
- **关联**: EK-43、EK-45、KO-07

## C-04 `IMPORT_TIMEOUT_SEC=10` 仅存在于配置，主流程未见强制超时
- **类型**: 项目内配置假说
- **当前证据**: `MemEvolve/config.py` 定义 `IMPORT_TIMEOUT_SEC=10`；`phase_validator.py:433-790` 的 import 调用未见 `signal.alarm`/线程超时。
- **缺失证据**: 未全文检索所有 `IMPORT_TIMEOUT_SEC` 使用点；可能由外部 runner 使用。
- **验证路径**: `grep -rn IMPORT_TIMEOUT_SEC` 并构造一个 import 时 sleep 20s 的 provider 观察是否被中断。
- **Epistemic**: Hypothesis · **影响**: 一个坏 provider 可挂起整轮进化（与 F7-E6 同源）。
- **关联**: EK-49、EK-44

## C-05 “11 基线 + 2 evolved”无法从代码映射到具体 provider
- **类型**: 文档-代码对齐假说
- **当前证据**: README 声明；`MemoryType` 13 项、`PROVIDER_MAPPING` 13 项；代码无 `is_evolved` 字段或注释标记。
- **缺失证据**: 论文附录/实验配置可能定义；仓库内未见。
- **验证路径**: 对照 arXiv 2512.18746 附录的 provider 清单与仓库 enum。
- **Epistemic**: Hypothesis（文档声明） · **影响**: 任何“evolved vs baseline”的对比结论都不能直接引用代码。
- **关联**: EK-03、E-093

## C-06 Cerebra 抽取的工具代码在真实任务中是否可安全执行，未验证
- **类型**: 运行时假说
- **当前证据**: 工具体系含抽取、安全检查、`wrapped_tool` 返回（E-050）；未见沙箱/资源限制。
- **缺失证据**: 未运行；未见工具执行失败率/注入攻击测试。
- **验证路径**: 构造含危险操作的轨迹，观察 `_is_dangerous_code` 是否拦截；统计工具调用成功率。
- **Epistemic**: Hypothesis · **影响**: API 型记忆的可靠性/安全性未知（KO-09 的边界）。
- **关联**: EK-26、EK-50、KO-09

## C-07 论文自报提升（最高 +17.06%）在本仓库状态下不可复现
- **类型**: 外部效度假说
- **当前证据**: README 自述 ICML'26、arXiv 2512.18746、多个基准提升（E-092/E-096）；仓库“not the final release”（E-094）。
- **缺失证据**: 未配置 API key、未下载完整数据、未跑 benchmark；论文实验配置可能依赖内部版本。
- **验证路径**: 按 README 命令在 GAIA/WebWalkerQA 上跑基座与 2 个 evolved provider，记录 accuracy/token；与论文表对照。
- **Epistemic**: Hypothesis（S0-vendor） · **影响**: 任何性能结论只能标注“作者自报”。
- **关联**: E-092~E-096

## C-08（跨项目）阶段感知的记忆注入可能比“更大记忆库”更有效
- **类型**: Cross-project Candidate
- **当前证据**: 7/13 provider 仅 BEGIN，5/13 双阶段；Cerebra 在 IN 阶段使用独立阈值与 context 拼接（E-019/E-018）；但本仓库无 ablation。
- **缺失证据**: 缺同一 provider “开/关 IN” 的对照实验；缺跨项目复现。
- **验证路径**: 固定存储与检索，仅切换 IN 注入开关，比较任务成功率/步数；在第二套 Agent 框架复现。
- **Epistemic**: Hypothesis（跨项目） · **禁止**: 不得写成 Principle。
- **关联**: KO-02、EK-07、EK-27

## C-09（跨项目）验证器-修复者分离度可作为质量门的独立指标
- **类型**: Cross-project Candidate
- **当前证据**: 本项目 auto-fix 默认开启且回写（E-067/E-068）；快检通过率无法区分“生成正确”与“被修正确”。
- **缺失证据**: 未统计修复率；未在其他 CI/审查系统测量。
- **验证路径**: 定义指标 `fix_rate = 修复后通过数 / 总候选数` 与 `raw_pass_rate`；在 ≥2 个项目上比较二者与下游缺陷率的相关性。
- **Epistemic**: Hypothesis（跨项目） · **禁止**: 不得写成已验证 Principle。
- **关联**: KO-07、EK-43、EK-45

## C-10（跨项目）自我修改系统需要“变异事务”抽象
- **类型**: Cross-project Candidate
- **当前证据**: Creator 顺序写文件/注册表/配置，无回滚（E-063/E-064）；失败留下半成品。
- **缺失证据**: 未观察半成品对下一轮生成的实际影响；未在其他 self-modifying agent 上复现。
- **验证路径**: 注入一个必然失败的 config_updates，检查 registry 是否残留；下一轮生成是否引用残留 provider。
- **Epistemic**: Hypothesis（跨项目） · **禁止**: 不得写成 Law。
- **关联**: KO-06、EK-39、EK-40

## C-11（独立验证补录）失败轨迹的摄取语义在 provider 间不一致
- **类型**: 项目内机制假说
- **当前证据**: 异常任务以 `is_correct=False` 调用 `take_in_memory`（`run_..._gaia.py:369-388`）；MemP 在失败时修订旧记忆（E-020）；Agent-KB 只在成功时写入（E-014）。
- **缺失证据**: 未逐 provider 核对失败分支是否写库；未测失败记忆对后续任务的影响。
- **验证路径**: 构造同一失败轨迹喂给 13 个 provider，统计各自 JSON 文件变化条数。
- **Epistemic**: Hypothesis · **影响**: “失败经验”在不同系统中或进入、或丢弃、或改写，跨系统比较失真。

## C-12（独立验证补录）Evolver 使用量反馈可能跨任务失效
- **类型**: 项目内缺陷假说
- **当前证据**: `_last_provided_cache` 为实例属性（`evolver:388-410`）；runner 每个线程创建独立 provider 实例（`run_..._gaia.py:594-607`）。
- **缺失证据**: 未确认线程是否复用实例；未观察多任务后 `usage` 是否增长。
- **验证路径**: 连续 10 个任务使用同一 Evolver 实例与每任务新实例各跑一次，比较 `usage_count`。
- **Epistemic**: Hypothesis · **影响**: 若成立，`(success+1)/(usage+2)` 退化为常数，原则库无法自我排序。

## C-13（独立验证补录）`take_in_memory` 失败无重试/无监控
- **类型**: 项目内可靠性假说
- **当前证据**: 返回值 False 只 warning（`run_..._gaia.py:336-338`、`386-388`）。
- **缺失证据**: 未统计发生率；无指标导出。
- **验证路径**: 注入只读目录，跑 100 任务，统计 warning 行数与记忆条数差。
- **Epistemic**: Hypothesis · **影响**: 长期运行记忆静默丢失（与 R-06 清理缺口同源）。

## C-14（独立验证补录）失败候选的隔离目录不清理，磁盘会累积
- **类型**: 项目内运维假说
- **当前证据**: 清理仅在 `verdict=passed` 分支（`phase_validator.py:804-810`）。
- **缺失证据**: 未测单轮失败候选的目录体积。
- **验证路径**: 连续跑 5 轮失败验证，统计 `EvolveLab` 下临时目录数量。
- **Epistemic**: Hypothesis · **影响**: 长跑后磁盘占满导致后续验证失败。
