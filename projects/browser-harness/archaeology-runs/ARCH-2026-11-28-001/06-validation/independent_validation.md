# 阶段二 · 冻结后对比验证报告（Validation Report）

- 审计对象：考古包 `package/`（00–06 + `run_metadata.yaml`），对照阶段一冻结产物 `audit/independent_findings.md`。
- 基准仓库：`repo/` @ `373aecc4c40ef67d97b5ee69c144684a64a42a11`（v0.1.15）；外部证据：PyPI wheel `browser-use==1.0.8`、`browser-harness==0.1.13/0.1.14`（`audit/_scratch/`）。
- 判定标签：CONFIRMED / PARTIALLY_CONFIRMED / DOWNGRADED / OVER_GENERALIZED / MISSING / CONTRADICTED / NEEDS_HUMAN_REVIEW。
- 三向对齐口径：**包原文（文件:行号）→ 独立发现编号（IF-*，见 independent_findings.md）→ 仓库证据 file:line**。
- 本轮未读 `audit/second_opinion_checks.md`：该文件不存在（`audit/` 初始为空，仅有本审计产出），无双人对照项。
- 计数口径见第 1 节；所有行数/函数数为本审计实测 `wc -l` / `grep -c`。

## 1. 判定统计表

### 1.1 判定项与计数

| 标签 | 条数 | 计数口径 |
|---|---:|---|
| CONFIRMED | 79 | 包内主张可在代码/测试/元数据定位，证据链闭合，范围未被夸大 |
| PARTIALLY_CONFIRMED | 7 | 主张主干成立，但存在锚点漂移、事件形态依赖或条件未闭合（EK-06/37/41/43、KO-04、C-05、C-07） |
| DOWNGRADED | 1 | 认识论层级被拔高，单项目证据不足以支撑所标层级（KO-09） |
| OVER_GENERALIZED | 1 | 函数级设计被升格为运行时路径上的通用行为（EK-10） |
| MISSING | 6（另列于第 4 节） | 仓库客观存在、包未覆盖的机制/路径；不计入既有主张的判定 |
| CONTRADICTED | 3 | 锚点在树内不可解析，或与实测计数/代码直接冲突（EK-56/OV-4 同一锚点、PL-5） |
| NEEDS_HUMAN_REVIEW | 3 | 代码路径可见但未运行验证，包自身已标 Hypothesis，保留人工复核（C-01/03/06） |
| **已判定主张合计** | **94** | 台账行：58 EK + 10 KO + 7 Flow + 8 Candidate + 11 条 00/01 头条主张；EK-56 与 OV-4 按同一锚点的两行分别计列 |

### 1.2 统计说明

- EK 58 条、KO 10 条、Flow 7 条、Candidate 8 条均逐一过账，明细见第 7 节台账。
- "MISSING" 不与既有主张互判：它表示包整体未覆盖的仓库事实，单列第 4 节。
- 锚点漂移（行号偏 2–15 行、符号仍可解析）不单独计 CONTRADICTED，计入 PARTIALLY_CONFIRMED 并在台账注明；锚点完全不可解析才计 CONTRADICTED。
- 本审计未执行测试、未启动浏览器/CDP；凡包内标注"未运行"的推断，本轮同样保持静态判定。

## 2. 三个成功（高质量、证据充分的结论）

### S-1 · owner-shared 只拒绝两个破坏性 helper，`cdp()`/`js()` 仍可驱动父 owner 当前页

| 方向 | 内容 |
|---|---|
| 包原文 | `02-ek` EK-11（`ek_graph.md:90-96`）；`03` KO-02（`knowledge_layer.md:44-62`）；`05` C-03（`candidates.md:24-31`）；`00` 行 16 |
| 独立发现 | IF-B（专项 B：成立）；IF-Q4-1 |
| 仓库证据 | 拒绝点仅 `helpers.py:445`（release_tabs）、`helpers.py:523`（close_tab），二者调 `_refuse_for_shared_owner`（`helpers.py:114-117`）；绕过路径：`cdp` `helpers.py:96-103`、`js` `helpers.py:684`、`goto_url` `:196`、`click_at_xy` `:234`、`switch_tab` `:501` 等均无 refuse/warn；测试反向佐证 `tests/unit/test_helpers.py:700-714`（设 `BH_OWNER=child` 后 close/release 才发出） |
| 为何高质量 | 代码路径、拒绝清单、绕过清单、测试四重闭合；包未把"可驱动父页"夸大成"可关闭父页"，边界表述准确（EK-11 scope 行 93）。 |

### S-2 · 协议不匹配时，只有"本地 + 空闲 + 旧协议 + 允许重启"四条同时满足才重启；newer/cloud/cdp/active/provisioned 一律 fail-closed

| 方向 | 内容 |
|---|---|
| 包原文 | EK-34（`ek_graph.md:278-284`）、EK-35（`:286-292`）；EV-34/EV-35（`evidence.md:41-42`）；`05` C-08 旁证 |
| 独立发现 | IF-D（专项 D：成立）；IF-Q3-2 |
| 仓库证据 | `admin.py:422-445` 全分支矩阵：无应答/同协议不动作（428-429）、newer 不停（430-431）、`may_restart=False`（432-433）、非 local（434-436）、`active>0`（437-438）、唯一重启分支（439-441）；调用方 `ensure_daemon` `admin.py:667`（默认允许）、`require_existing_daemon` `admin.py:880`（禁止）；测试 `tests/unit/test_admin.py:218-250`（断言 provisioned 分支重启函数零调用） |
| 为何高质量 | 分支条件、调用方差异、报错语义（"Nothing was sent"）与测试矩阵一一对应；包未把 fail-closed 描述成"自动升级"。 |

### S-3 · browser-use 1.0.8 钉死 `browser-harness==0.1.13`，自身不消费 owner API；共享 `Browser('chrome')` 不带 owner 身份

| 方向 | 内容 |
|---|---|
| 包原文 | EK-58（`ek_graph.md:476-482`）；C-08（`candidates.md:69-76`）；`00` 行 18；EV-58（`evidence.md:65`） |
| 独立发现 | IF-E（专项 E：成立）；IF-Q5-1、IF-Q5-2 |
| 仓库/外部证据 | `bu108/browser_use-1.0.8.dist-info/METADATA:21` `Requires-Dist: browser-harness==0.1.13`；请求构造不含 owner 字段 `bu108/browser_use/browser/browser.py:350-356`；daemon 名回退链 `browser.py:1036-1037`；worker 仅复用 `current_tab/switch_tab/cdp/_send/http_get/wait_for_network_idle`（`worker.py:324-334,496-536,677-711`）；0.1.13 wheel `daemon.py` 对 owner/session_owner/PROTOCOL 零命中；非共享 Browser 唯一名 `bu{uuid4().hex[:8]}` `browser.py:940` 一带 |
| 为何高质量 | 版本钉死有元数据实证，"零 owner API"有全量 grep 与 worker 调用清单实证；包正确区分了"事实（钉版/无 API）"与"推断（用户影响）"，未把影响写成已验证结论。 |

## 3. 三个错误（代表性 CONTRADICTED / OVER_GENERALIZED / DOWNGRADED）

### E-1 · CONTRADICTED：被反例对照的 README 原句在本快照树内不存在

| 方向 | 内容 |
|---|---|
| 包原文 | EK-56（`ek_graph.md:460-466`，源引 `README.md`）；EV-56（`evidence.md:63`，源引 `README.md`）；`00` 行 16："The README's sentence 'Isolation and privacy hold for raw CDP too'…"；F-05 边界 `flow_atlas.md:97` |
| 独立发现 | IF-Q4-2；IF-A |
| 仓库证据 | 对 `repo/**/*.md` 全量 grep `raw CDP` / `Isolation and privacy` / `privacy hold` / `isolation`：**零命中**该句；`README.md` 全文（137 行）无"raw CDP 隔离/隐私"宣称；`SKILL.md:52` 仅写 "Raw CDP is available with `cdp(...)`"，`interaction-skills/local-chrome.md:11` 谈的是另一回事。 |
| 错在哪 | 代码侧结论（browser-level 会话不绑 owner）本身成立（见 S 类与 IF-A），但"文档做了隔离宣称、代码构成反例"这一**对照锚点**在树内不可解析。包把一条无法定位的文档引述当成了事实来源，违反其自定规则"single-source row is an Observation"。 |
| 影响 | 中等：结构判断未被推翻，但 EK-56/EV-56 的证据链缺一环；若上游 main 分支 README 确有此句，属快照与上游差异，需人工确认引用来源，不能在本树内当作已核事实。 |

### E-2 · OVER_GENERALIZED：`_warn_shared_owner` "每次调用都告警"被从函数设计升格为 CLI 运行时行为

| 方向 | 内容 |
|---|---|
| 包原文 | EK-10（`ek_graph.md:82-88`）："`_warn_shared_owner` prints a warning on every call, not once, because siblings share the id."；EV-10（`evidence.md:17`） |
| 独立发现 | IF-B（边界部分） |
| 仓库证据 | 函数本身确无 once 标志（`helpers.py:120-124`，docstring 亦称 "Said on every call"，测试 `test_helpers.py:717-730` 连调两次期望两条告警）；**但生产接线只有一处**：CLI 启动期 `run.py:471` 调用一次，`exec(code, globals())` 之后不再按 helper 调用触发；REPL worker（bu108 `worker.py`）未调用该函数。 |
| 错在哪 | 包把"函数可被反复调用、每次都打告警"的设计意图，写成了运行时路径上"每次调用都告警"的行为事实。实际 CLI 进程内只告警一次。 |
| 影响 | 低-中：不影响安全结论（拒绝逻辑仍只在两个 helper），但告警频次的描述与接线不符；属 Epistemic 越级（Observation 包装成 Fact 语气）。 |

### E-3 · DOWNGRADED：KO-09 以单项目证据标 L4 Cognitive Model，层级拔高

| 方向 | 内容 |
|---|---|
| 包原文 | KO-09（`knowledge_layer.md:184-202`，标 `L4 Cognitive Model`，confidence medium-high）；`03` 头行 3 称 "2 L4"；`run_metadata.yaml:44` `ko_l4: 2` |
| 独立发现 | IF-Q4-1、IF-A；认识论校准（L4 需跨案例心智模型，单仓库只够 L3 Pattern + 待证 Hypothesis） |
| 仓库证据 | KO-09 的全部 derivation.facts 为 EK-14/16/19/56，全部来自本仓库；`does_not_apply_when` 所列 browser CDP / 容器 / 数据库池均无第二案例证据；包自身 scope 行 187 已写 "single-project evidence; cross-project validation pending"，但层级标签仍落在 L4。 |
| 错在哪 | 单项目 + 单下游对比例子，不足以支撑跨域心智模型；正确层级应为 L3 Pattern（本仓库内成立）+ 跨项目 Hypothesis（对应 C-06）。包的 caveat 部分缓解了问题，但未纠正层级标签。 |
| 影响 | 中：L4 会被下游复用为"通用授权分析模型"；当前证据只支持"本 broker 存在可见面与可达面之差"。另注：`ko_l4=2` 与正文不符（见第 6 节计数复核）。 |

## 4. MISSING：包未覆盖但仓库客观存在的机制/路径

| # | 遗漏项 | 仓库证据 | 重要性 |
|---|---|---|---|
| M-1 | **0.1.15 遥测 cli_event 载荷瘦身**（相对 0.1.14 的未记录变更）：不再出域脚本文本 `task`、stdout/stderr 尾 `output`、helper 调用参数与异常文本；步骤仅记 helper 名/时长/错误布尔，异常归类为最近内置类名；agent_helpers 名归 `custom`；stderr 不再被 `_StreamTail` 流式包装截取（0.1.15 仅 `_CountingStream` 计字符数），配套删除 `MAX_TASK_LENGTH`/`_MAX_STEP_ARGS_LENGTH`/`_MAX_OUTPUT_LENGTH`/`_StreamTail`。CHANGELOG 无 0.1.15 段，包亦未识别此 delta。 | `run.py:207-237,260,323-356`；`telemetry.py:282-315`；`helpers.py:843`；对照 `_scratch/bh0114/`：`run.py:207,219,253,256,313-314`、`telemetry.py:22,313` | 高：属发布事实与隐私边界，Q5/F 专项要求覆盖 |
| M-1b | **显式 `close_tab` 绕过"最后真实页"保护**：meta `close_tab` 只过 `_mine()` 归属校验即 `closeTarget`，不做释放路径的 last-real-page 判断；可关闭全局最后一个真实页。 | `daemon.py:1144-1152`（对照保护判断 `:737`） | 中：属 alternate path，包内 KO-04 不变量未声明此例外 |
| M-2 | **owner 持久化文件的权限面**：`bu-<name>.owners.json` 存 target id/current/marks/created/name，`_save` 用 tmp+replace 但无显式 0600 chmod；同用户进程可读可写。包仅在候选 C-05 提及，EK 层无对应节点。 | `daemon.py:635-643`；目录 0700 见 `paths.py:19-23` | 中：与 C-01 组合后是句柄泄露面 |
| M-3 | **嵌入方对 helper 的运行时包装/替换**：browser-use worker 把 `cdp/http_get/wait_for_network_idle/switch_tab` 等在命名空间内 monkeypatch 包装；该旁路/覆盖路径未进 Flow 与 EK 图。 | `bu108/.../worker.py:496-536,695-707` | 中：属于 "alternate path / override" 审计要点 |
| M-4 | **外围域未建节点**（包自承未逐行）：`video.py` 751 行、`video_render.py` 525 行、`macos.py` 196 行、`auth.py` 569 行、`recorder.py` 自动录音状态机（`observe/auto_start/staleness` 228-242）。 | 实测 `wc -l`；`recorder.py:142-242` | 中：范围声明诚实，但知识库缺这些面的存在性记录 |
| M-5 | **入口与 MCP 工具面**：`naming.py`（CLI 名别名）、`mcp_cli.py` 懒加载、`mcp_server.py` 26 个工具经由 helpers 继承 OWNER 的路径，未建 EK/Flow 节点。 | `naming.py:1-16`；`mcp_cli.py:1-16`；`src/mcp_server.py:161-334` | 低-中 |
| M-6 | **0.1.13 运行时下不可用能力清单**：owner 归属、协议握手、`release_tabs/list_browsers/use_browser/name_session/mark_tab/solve_captcha` helper、browser-session 下载归属、idle exit、cdp-mismatch 等仅在 EK-58 结论中笼统带过，未逐项列明。 | 对照 `_scratch/bh0113/browser_harness/helpers.py` 函数清单与 `daemon.py` 零命中；0.1.15 对照 `helpers.py:441-590`、`daemon.py:509-513` | 中：版本缝隙的可操作性证据 |

## 5. Benchmark case（可复用回归/基准审计用例）

### 用例：browser-level CDP 会话的 sid 归属校验缺口

- **输入**：一份被审考古包中关于"raw CDP 隔离由 daemon 归属表保证"的主张（对应包内 EK-17/EK-19/F-05 一类条目）。
- **期望判定要点**（真读代码者必须能答出）：
  1. `Target.attachToBrowserTarget` 返回的 sid 只写入 `browser_sessions`，**不**写 `session_owner`（`daemon.py:1220-1221`）；
  2. 两处显式 sid 校验均为 `session_owner.get(sid, key) != key`，**默认值等于当前调用方 key**，未登记 sid 默认放行（`daemon.py:1181,1233`）；
  3. 显式 sid 分支不做 stale-session 重挂重试；未知 sid 由 Chrome 报错上抛，不能借此驱动他者已存在会话（`daemon.py:1235` vs 重试分支 `1246-1252`）；
  4. 事件方向对 browser session 另有 frameId/guid 兜底路由，与命令方向校验不对称（`daemon.py:826-833`）。
- **区分力**：上述四点均不在 README/SKILL/CHANGELOG 等文档中；只读文档的审计者只能复述"有 owner 隔离"的宣称，无法指出默认值放行这一具体机制，也分不清命令面与事件面的不对称。该用例同时覆盖 bypass/alternate path 与 Epistemic 状态核查，可回归检验后续考古包是否真读了 `daemon.py` 的 `_send`/`_route_event`。
- **预期标签**：代码路径 CONFIRMED；可利用性 NEEDS_HUMAN_REVIEW（sid 是否暴露给第二 owner 未运行实测）。

## 6. `run_metadata.yaml` 质量指标独立复核

| 指标（包内值） | 本审计实测 | 结论 |
|---|---:|---|
| `src_python_lines: 8428` | `find src -name '*.py' \| wc -l` = 8428 | ✅ 站得住 |
| `test_python_lines: 6830` | tests 合计 6356（unit）+ 437（integration）= 6830 | ✅ 站得住 |
| `test_functions: 404` | `grep -rh "def test_" tests/ \| wc -l` = 404（unit 387 + integration 17） | ✅ 站得住 |
| `tracked_files: 299` | `git ls-files \| wc -l` = 299 | ✅ 站得住 |
| `ek_nodes: 58` | `grep -c "^### EK-"` = 58 | ✅ 站得住 |
| `candidates: 8` | `grep -c "^## C-"` = 8 | ✅ 站得住 |
| `evidence_rows: 58` | `grep -c "^| EV-"` = 58 | ✅ 站得住 |
| `ko_l3: 8` / `ko_l4: 2` | 正文实测：KO-01…08、10 为 L3（**9 条**），KO-09 为唯一 L4（**1 条**）；`knowledge_layer.md:3` 与 `self_validation.md:12,52,74` 自相矛盾（一处称 2 L4，一处写 "L4 only for KO-09"） | ❌ **站不住**：应为 9 L3 / 1 L4，计数与层级标签需修正 |
| `ek_edges: 144` / `avg_out_degree 2.48` / `orphans 0` | 本审计仅确认 58 条 `links:` 行，未解析边串复算边数 | ⚠️ 无法独立复核；建议复跑边解析脚本后再采信 |
| `distinct_anchors: 129` / `dead_anchors_target_repo: 0` | 抽样 20+ 锚点：多数符号可解析但存在行号漂移（见 7.2 PARTIAL 项）；EV-56 的 README 锚点在树内不可解析（E-1） | ❌ **"0 dead anchors" 站不住**：至少 1 条文档锚点失效，另有多条行号区间漂移 |
| `core_modules_inspected: 12/17` | 与自承范围一致（video/auth/macos/recorder 等外围未逐行）；但 EK 图对 recorder 仅经 EK-02 间接提及，存在"已检查模块"与"已建知识节点"不等价 | ⚠️ 部分成立 |
| `validation.self.status: CONDITIONAL_PASS`；`contradictions: 0` | 本审计发现至少 2 处包内问题：README 锚点不可解析（E-1）、L3/L4 计数矛盾（第 6 行）；self-validation 的 "contradictions 0" **站不住** | ❌ 需下调；开放检查四条（browser-level 未执行、info freshness、video/macOS 未覆盖、tests 未跑）与本审计独立发现一致，✅ 诚实 |
| `tests_executed: false` / `runtime_probes: 0` | 与实测一致，包未谎称运行过 | ✅ 站得住 |

## 7. 台账：逐主张三向对齐

### 7.1 EK 节点（58）

| ID | 包原文锚点 | 标签 | IF 编号 | 仓库证据 file:line | 备注 |
|---|---|---|---|---|---|
| EK-01 | `ek_graph.md:8-14` | CONFIRMED | IF-Q3-1 周边 | `run.py:276-286,472-474` | |
| EK-02 | `:16-22` | CONFIRMED | IF-F | `run.py:202-237`；参数不入 trace、recorder 本地另收 `run.py:237` | 未识别为 0.1.15 未记录 delta（见 M-1） |
| EK-03 | `:24-30` | CONFIRMED | IF-F | `helpers.py:828-844,843` | |
| EK-04 | `:32-38` | CONFIRMED | IF-Q3-1 | `daemon.py:503-522,1276-1300` | |
| EK-05 | `:40-46` | CONFIRMED | IF-Q1-1 | `helpers.py:41,75-88`；`_ipc.py:130-150` | |
| EK-06 | `:48-54` | PARTIALLY_CONFIRMED | IF-Q3-2 | `_ipc.py:30`（PROTOCOL=2；包引 `:47` 漂移）；`admin.py:422-446` | 主张成立，锚点行号错 |
| EK-07 | `:56-62` | CONFIRMED | IF-Q1-3 | `daemon.py:503-505`；`run.py:25-47` | |
| EK-08 | `:64-70` | CONFIRMED | IF-Q3-3 | `daemon.py:1350-1360` | |
| EK-09 | `:74-80` | CONFIRMED | IF-Q1-1 | `_ipc.py:33-36` | |
| EK-10 | `:82-88` | OVER_GENERALIZED | IF-B | `helpers.py:120-124`；接线仅 `run.py:471`；测试 `test_helpers.py:717-730` | 见 E-2 |
| EK-11 | `:90-96` | CONFIRMED | IF-B | `helpers.py:114-118,445,523`；绕过 `:96-103,684` | 见 S-1 |
| EK-12 | `:98-104` | CONFIRMED | IF-Q1-2 | `daemon.py:608-617,619-633` | |
| EK-13 | `:106-112` | CONFIRMED | IF-Q1-2 | `daemon.py:487-507` | |
| EK-14 | `:114-120` | CONFIRMED | IF-Q4-1 | `daemon.py:43,1210-1219` | |
| EK-15 | `:122-128` | CONFIRMED | IF-Q4-1 | `daemon.py:566-576,1215-1218` | |
| EK-16 | `:130-136` | CONFIRMED | IF-Q4-3 | `daemon.py:556-564`；测试 `test_daemon.py:484`（包引 533-556 漂移） | |
| EK-17 | `:138-144` | CONFIRMED | IF-Q4-1 | `daemon.py:817-831,1219-1223` | |
| EK-18 | `:146-152` | CONFIRMED | IF-Q2-1 | `daemon.py:1239-1253` | |
| EK-19 | `:154-160` | CONFIRMED（代码路径） | IF-A | `daemon.py:530,1220-1221,1233`；可利用性未测 |  exploitability 属 Hypothesis，包标注正确 |
| EK-20 | `:164-170` | CONFIRMED | IF-Q2-1 | `daemon.py:662-671`；测试 `test_daemon.py:289-299` | |
| EK-21 | `:172-178` | CONFIRMED | IF-Q2-1 | `daemon.py:666-671` | |
| EK-22 | `:180-186` | CONFIRMED | IF-Q2-1 | `daemon.py:1192-1197`；`helpers.py:510-518` | |
| EK-23 | `:188-194` | CONFIRMED | IF-Q2-1 | `daemon.py:852-854` | |
| EK-24 | `:196-202` | CONFIRMED | IF-C | `daemon.py:726-750`；测试 `test_daemon.py:970-986` | |
| EK-25 | `:204-210` | CONFIRMED | IF-Q2-2 | `daemon.py:752-770`；测试 `test_daemon.py:1071-1085` | |
| EK-26 | `:212-218` | CONFIRMED | IF-C | `daemon.py:736-738`；测试 `test_daemon.py:1016-1028,1182-1193` | |
| EK-27 | `:220-226` | CONFIRMED | IF-Q1-2 | `daemon.py:153,1123-1134`；`helpers.py:471-475` | |
| EK-28 | `:228-234` | CONFIRMED | IF-Q2-1 | `daemon.py:1144-1152`；测试 `test_daemon.py:557-568` | |
| EK-29 | `:236-242` | CONFIRMED | IF-Q2-1 | `daemon.py:1246-1252`；测试 `test_daemon.py:784-810` | |
| EK-30 | `:244-250` | CONFIRMED | IF-Q1-2 | `daemon.py:156-180,712-724` | |
| EK-31 | `:254-258` | CONFIRMED | IF-Q3-1 | `admin.py:391-411`；`_ipc.py:151-170` | |
| EK-32 | `:262-268` | CONFIRMED | IF-Q3-3 | `_ipc.py:173-210`（`type(pid) is int`，`0 < pid < 2**31`）；`admin.py:904-1040` | |
| EK-33 | `:270-276` | CONFIRMED | IF-Q3-1 | `admin.py:279-336,700-720` | |
| EK-34 | `:278-284` | CONFIRMED | IF-D | `admin.py:422-445` | 见 S-2 |
| EK-35 | `:286-292` | CONFIRMED | IF-D | `daemon.py:1072-1073`；`admin.py:437-438` | |
| EK-36 | `:294-300` | CONFIRMED | IF-Q3-1 | `admin.py:587-620`；`daemon.py:286-300`（`_cdp_keys`） | |
| EK-37 | `:302-308` | PARTIALLY_CONFIRMED | IF-Q3-1 | 标记读写 `_ipc.py:63-87`；fail-closed 检查 `admin.py:682-691`（包引 760-790 漂移） | 主张成立，锚点漂移 |
| EK-38 | `:310-316` | CONFIRMED | IF-Q3-3 | `admin.py:192-278,904-1040` | |
| EK-39 | `:320-326` | CONFIRMED | IF-Q1-4 | `_ipc.py:246-262`；`paths.py:19-23` | |
| EK-40 | `:328-334` | CONFIRMED | IF-Q1-4 | `_ipc.py:263-278`；`daemon.py:1057-1063`；端口文件模式包内已存疑 | |
| EK-41 | `:336-342` | PARTIALLY_CONFIRMED | — | 常量 `_ipc.py:27`（`32 << 20`）；读取超限 `:208-226`（包引 39-44 漂移）；响应无上限未在该文件见到反证 | |
| EK-42 | `:344-350` | CONFIRMED | IF-Q4-1 | `daemon.py:694-710`；测试 `test_daemon.py:1205-1219` | |
| EK-43 | `:352-358` | PARTIALLY_CONFIRMED | IF-Q4-3 | `daemon.py:829-833`；归属依赖 Chrome 事件带 frameId/guid，包标 medium 合理 | |
| EK-44 | `:362-368` | CONFIRMED | IF-Q2-1 | `daemon.py:853-858` | |
| EK-45 | `:370-376` | CONFIRMED | IF-Q2-1 | `daemon.py:619-633,847-852` | |
| EK-46 | `:378-384` | CONFIRMED | IF-Q4-3 | `daemon.py:867-909,912-959` | |
| EK-47 | `:386-392` | CONFIRMED | IF-Q1-2 | `daemon.py:635-660` | |
| EK-48 | `:394-400` | CONFIRMED | IF-Q2-1 | `daemon.py:645-660` | |
| EK-49 | `:402-408` | CONFIRMED | IF-Q2-2 | `daemon.py:164-176,752-775`；`IDLE_EXIT` `:146` | |
| EK-50 | `:410-416` | CONFIRMED | IF-Q3-3 | `daemon.py:1350-1360` | |
| EK-51 | `:418-424` | CONFIRMED | IF-Q3-3 | `daemon.py:1302-1341` | |
| EK-52 | `:426-432` | CONFIRMED | IF-Q3-3 | `daemon.py:1362-1378,1079-1086` | |
| EK-53 | `:436-442` | CONFIRMED | IF-Q4-3 | `report.py:134-160`；caps `:33-34` | |
| EK-54 | `:444-450` | CONFIRMED | IF-Q4-3 | `telemetry.py:22-40,149-163` | |
| EK-55 | `:452-458` | CONFIRMED | — | `helpers.py:771-787` | |
| EK-56 | `:460-466` | CONTRADICTED | IF-A | 代码侧缺口见 `daemon.py:1220-1221,1233`；所引 README 句在树内不可解析 | 见 E-1 |
| EK-57 | `:468-474` | CONFIRMED | — | `AGENTS.md:23`；`SKILL.md:2`；`pyproject.toml:6` | |
| EK-58 | `:476-482` | CONFIRMED（事实部分） | IF-E | `bu108 METADATA:21`；`browser.py:350-356,1036-1037,940`；影响推断包标 medium | 见 S-3 |

### 7.2 KO / Flow / Candidate / 头条主张

| ID | 包原文锚点 | 标签 | IF 编号 | 仓库证据 | 备注 |
|---|---|---|---|---|---|
| KO-01 | `knowledge_layer.md:24-42` | CONFIRMED | IF-Q1-4 | 聚合 EK-12/14/15/16/17/18/28/29 均闭合；反例 EK-19/EK-11 已记 | L3 层级恰当 |
| KO-02 | `:44-62` | CONFIRMED | IF-Q1-4、IF-B | EK-09/10/11/14/18 闭合 | |
| KO-03 | `:64-82` | CONFIRMED | IF-Q2-2 | EK-13/20/24/25/26 闭合 | |
| KO-04 | `:84-102` | PARTIALLY_CONFIRMED | IF-C | 不变量代码成立；`last` 计数依赖 `self.info` 缓存、无新鲜交叉校验（`daemon.py:737`）；info 滞后为开放反例，包已标注 | |
| KO-05 | `:104-122` | CONFIRMED | IF-Q2-1 |  typed 错误集合 `daemon.py:590-596,604-606` | |
| KO-06 | `:124-142` | CONFIRMED | IF-Q3-1/Q3-3 | EK-31/32/33/34/35/36/38 闭合 | |
| KO-07 | `:144-162` | CONFIRMED | IF-Q3-1 | EK-36/37/49/50 闭合 | |
| KO-08 | `:164-182` | CONFIRMED | IF-Q4-3 | EK-53/54/55 闭合；反例（key filter 非文本 scrub）已记 | |
| KO-09 | `:184-202` | DOWNGRADED | IF-Q4-1 | 单仓库证据推 L4；应回落 L3 + 跨项目 Hypothesis | 见 E-3 |
| KO-10 | `:204-222` | CONFIRMED | IF-Q5-1/Q5-3 | EK-07/09/19/58 闭合；L3 层级恰当 | |
| F-01 | `flow_atlas.md:6-28` | CONFIRMED | IF-Q4-1 | 边符号均可解析；`handle` 实际 `daemon.py:1025`、`_handle:1057`（包引 1070-1096 漂移） | |
| F-02 | `:30-46` | CONFIRMED | IF-Q2-1 | 状态迁移闭合；`owner()` 实际 `daemon.py:532-535`（包引 545-548 漂移） | |
| F-03 | `:48-63` | CONFIRMED | IF-Q4-3 | `_visible:556-564`；请求 32 MiB 上限不约束响应（`_ipc.py` 无响应上限） | |
| F-04 | `:65-79` | CONFIRMED | IF-F | `run.py:215-237`；`report.py:320,328`；caps `report.py:33-34` | |
| F-05 | `:81-97` | CONFIRMED | IF-A | browser session 登记实际 `daemon.py:1221`（包引 1226-1228 漂移）；无 owner 绑定成立 | |
| F-06 | `:99-112` | CONFIRMED | IF-Q1-2 | 事件环 500 `daemon.py:491`；captcha 60s `:41,924` | |
| F-07 | `:114-126` | CONFIRMED | IF-Q2-2/Q3-2 | 重启策略每次调用重估、无持久策略对象，与代码一致 | |
| C-01 | `candidates.md:6-13` | NEEDS_HUMAN_REVIEW | IF-A | 代码路径确认；双 owner 跨会话读 cookie 探针未执行 | 包状态标注正确 |
| C-02 | `:15-22` | CONFIRMED | IF-Q1-3 | 旧"无 mutex" framing 被降级正确；协作式非认证边界成立 | |
| C-03 | `:24-31` | NEEDS_HUMAN_REVIEW | IF-B | 代码路径确认；真实 Claude 子代理宿主行为未测 | |
| C-04 | `:33-40` | CONFIRMED（Hypothesis 级） | — | `helpers.py:828-844` 公共名覆盖 core；无测试，包已标 | |
| C-05 | `:42-49` | PARTIALLY_CONFIRMED | M-2 | 目录 0700；`_save` 无显式 chmod（`daemon.py:635-643`）；同用户可读写为推断 | |
| C-06 | `:51-58` | NEEDS_HUMAN_REVIEW | — | 对照语料（deepseek-harness/opencode）本轮未重读；包标 Cross-project Hypothesis 正确 | |
| C-07 | `:60-67` | PARTIALLY_CONFIRMED | — | `run.py:25-47` 别名逻辑支持推断；双 BU_NAME 双 Allow 提示未实测 | |
| C-08 | `:69-76` | CONFIRMED（事实部分） | IF-E | 钉版/无 API 事实闭合；用户影响为推断，包已标 | |
| OV-1 | `00_overview.md:14` | CONFIRMED | IF-Q1-2/Q2-1 | 单 owner、三方法守护、url/title 清空、空闲回收、状态持久化均闭合 | |
| OV-2 | `:16` 前半 | CONFIRMED | IF-Q1-4/IF-B | 协作式非认证；仅 close/release 被拒；cdp 可驱动父页 | |
| OV-3 | `:16` 中段 | CONFIRMED（限定范围） | IF-A | browser session 未登记、默认值放行；浏览器域方法越出 tab 归属 | |
| OV-4 | `:16` 末段 | CONTRADICTED | IF-A | README 引述句树内不存在 | 同 E-1 |
| OV-5 | `:18` | CONFIRMED | IF-E | 钉 0.1.13、无 owner API、唯一名/共享 default 均实证 | |
| PL-1 | `project_layer.md:7-11` | CONFIRMED | IF-Q3-1 | 三平面划分与代码一致 | |
| PL-2 | `:25` | CONFIRMED | IF-Q1-2 | Owner 字段清单与 `daemon.py:487-507` 一致 | |
| PL-3 | `:37` | CONFIRMED | IF-Q2-1 | 调用方/daemon/tab 生命周期闭合 | |
| PL-4 | `:43` | CONFIRMED | IF-Q2-2 | 环境变量阈值与 `daemon.py:144-146,175,771` 一致 | |
| PL-5 | `:50` | CONTRADICTED | — | 实测 unit 测试文件 16、integration 4；包称 "17 unit files"，计数多 1 | 低影响计数错误 |
| PL-6 | `:56` | CONFIRMED | IF-Q4-1 | GUARDED 三方法，不含 browser-target/browser-domain | |

## 8. 攻击要点逐项结论

- **单案例 → Pattern**：未发现把单一代码路径升格为通用模式的硬伤；KO-01/02/03 均带 scope 与 does_not_apply_when。唯一越界是 KO-09 的 L4 拔高（E-3）。
- **Pattern → L4**：KO-09 为唯一 L4，证据仅单项目；其余 9 个 KO 保持 L3，层级纪律整体合格。
- **ADR/README/CHANGELOG/注释 → 实现事实**：发现 1 处文档锚点不可解析（E-1）；其余 EK 节点均回指代码行，未把文档宣称当代码行为。
- **Flow Edge 真实性**：7 条 Flow 的关键边均可在代码找到 symbol/condition/state transition；存在 5 处行号漂移（符号未失效），无凭空边。
- **bypass / override / exception / alternate path / direct CDP / admin path / fallback / legacy path**：包识别了子 agent `cdp()` 旁路（EK-11）、旧协议 `set_session` 兼容路径（`daemon.py:1169-1191`）、provisioned 不重启分支；**遗漏**了嵌入方 worker 对 helper 的 monkeypatch 包装（M-3）。
- **Epistemic 状态混淆**：EK-10 把函数设计写成运行时事实（E-2）；EV 登记表对单源码锚点的 Observation 标注整体诚实（`evidence.md:67-71`）。
- **跨项目假说写成已验证结论**：C-06 保持 Cross-project Hypothesis，未升入 03 层；KO-10 限定为一对上下游，未泛化。
- **Counterexample 被抹平**：KO-01/02/04/05/08/09/10 均记录了反例；未发现反例被删除或弱化。
- **自评全绿但证据链断裂**：self-validation 标 `contradictions: 0` 与实测不符（第 6 节）；其开放检查四条诚实，未掩盖未运行测试的事实。

## 8A. 与 `second_opinion_checks.md` 的双人对照

> 该文件在阶段一结束后出现，阶段二按规则读取。下表逐条对照；冲突项回仓库复核代码定案。

| 复核项 | 第二复核者判定 | 本审计判定（IF 编号） | 对照结论 | 最终依据 / 差异说明 |
|---|---|---|---|---|
| 断言 1：browser session sid 未登记、`get(sid,key)` 默认放行 | 成立 | IF-A：部分成立（机制成立，暴露面限定） | **双人一致（机制层）** | 登记点 685/823/1187/1223、校验点 1181/1233、`browser_sessions` 530/826/1221 双方一致。差异仅在措辞粒度：本审计强调未知 sid 在 Chrome 侧报 "Session with given id not found"、显式 sid 分支无 stale 重试（`daemon.py:1235,1246-1252`），故可利用面限于有效浏览器会话；不构成结论冲突。第二复核者补充的删除点 `daemon.py:626,863` 与 `if sid and ...` 短路（1181）已回查属实。 |
| 断言 2：共享 owner 仅拒 close_tab/release_tabs，驱动类 helper 绕过 | 成立 | IF-B：成立 | **双人一致** | 拒绝点 445/523、警告点 `run.py:471`、绕过清单（cdp 96-103、js 684 等）双方一致；测试 `test_helpers.py:700-714,717-730` 佐证一致。 |
| 断言 3：释放三态；默认不关；last 仅靠 self.info | 部分成立 | IF-C：部分成立 | **双人一致** | 三态 idle/forget/done 关闭范围、`BH_CLOSE_IDLE_TABS` 开关、`self.info` 单点判定、无释放期 getTargets 复核，双方一致。第二复核者明确"默认不关"仅对 idle/forget 成立，done 默认关闭自开未标记非最后真实页——本审计 IF-C 同口径（`daemon.py:736-738`）。 |
| 断言 4：outdated 四分支不重启；调用方；错误文本 | 成立 | IF-D：成立 | **双人一致** | 分支矩阵 `admin.py:422-445`、调用方 667(True)/880(False)、`active` 口径 `daemon.py:1072-1073` 一致。补充项 `keep` 标志禁止 idle 退出（`daemon.py:516,1071,773`）与 `restart_daemon` 只停不启（`admin.py:904-1003`）已回查属实，本审计 M-2/Q3-3 已覆盖 keep 路径。 |
| 断言 5：1.0.8 钉 0.1.13、零 owner API、共享 chrome 不带 owner | 成立 | IF-E：成立 | **双人一致** | METADATA 钉版、`browser.py:350-356` 请求无 owner、`browser.py:1036-1037` 回退 default 一致。补充：`legacy.py:468` 的方法名属 0.x 移除清单非调用；REPL worker 经 `helpers._send` 注入 OWNER（`helpers.py:81`），与本审计 M-3 一致。 |
| 断言 6：0.1.15 未记录变更（cli_event 隐私收紧） | 成立（7 条） | IF-F：成立（4 条主项） | **双人一致，第二复核者清单更全** | 双方均确认 CHANGELOG 无 0.1.15 段、wheel diff 仅 helpers/run/telemetry 三文件。第二复核者补充两条本审计未单列：⑥ stderr 不再被 `_StreamTail` 截取（0.1.14 `run.py:313`；0.1.15 `run.py:260` `_CountingStream` 仅计字符数）；⑦ 配套删除 `MAX_TASK_LENGTH`/`_MAX_STEP_ARGS_LENGTH`/`_MAX_OUTPUT_LENGTH`/`_StreamTail`。已回仓库复核：HEAD grep 四者零命中，0.1.14 wheel 均在（`bh0114/run.py:207,219,253,256,313-314`；`telemetry.py:22,313`）。本审计 M-1 据此补全。 |

### 8A.1 第二复核者旁路清单中、本审计回查后采纳的补充项

| 编号 | 旁路 | 仓库证据 | 本审计处置 |
|---|---|---|---|
| B7 | 显式 `close_tab` 只过 `_mine()`，不走"最后真实页"保护，可关掉全局最后一个真实页 | `daemon.py:1144-1152`（无 737 的 last 判断） | 采纳，补入 MISSING/边界：阶段一 IF-Q2-1 未单列此例外；属"释放路径保护不覆盖显式关闭"的 alternate path |
| D1 | 未知会话事件直接丢弃（fail-closed） | `daemon.py:834-835` | 采纳，与 IF-Q4-3 事件路由一致，包内 F-06 未明确此丢弃边 |
| E3 | `activate_tab` 裸 `Target.activateTarget`：GUARDED holder 校验通过即可，无主标签（holder None）可被激活而不接管归属 | `helpers.py:491-499`；GUARDED `daemon.py:43`；校验 `daemon.py:1213` | 采纳为条件性减弱路径；未升格为漏洞（设计上激活不等于接管） |
| A4 | GUARDED 仅 3 个 `Target.*` 方法；未列入的 `Target.detachFromTarget`/`setAutoAttach`/`createTarget`、`Browser.*` 不经 `_holder` 直达 | `daemon.py:43`；`daemon.py:1210-1219` | 采纳，补充 IF-Q4-1 的边界清单 |

### 8A.2 未决事项与第二复核者一致

- A2/A3 浏览器会话命令面可利用性、`self.info` 陈旧误判概率、1.0.8 REPL 运行时实际 owner：均未动态验证，保持 NEEDS_HUMAN_REVIEW。
- tag `v0.1.15` ↔ commit `373aecc` 的 GitHub 侧映射：REST API 限流未独立复核；旁证为本树 HEAD、PyPI 0.1.15 wheel 与本地 `src/` 逐文件一致（第二复核者已 diff 确认，本审计未重复该 wheel 比对）。

## 9. 未决与限制

1. 本审计全程静态：未执行 `tests/unit`、未启动 Chrome/CDP；C-01/C-03/C-07 的运行时行为保持 NEEDS_HUMAN_REVIEW。
2. `ek_edges=144`、`distinct_anchors=129` 两项需复跑脚本解析边串/锚点清单后才能复核，本报告不替包补算。
3. E-1 若上游 main 分支 README 确含该引述句，需核对快照分支差异；在本树内按不可解析处理。
4. 阶段一独立发现未因本轮对比回改；如有修正，按冻结规则应追加至 `independent_findings.md` 的 Errata 段——本轮未发现需要推翻阶段一结论的事项。
