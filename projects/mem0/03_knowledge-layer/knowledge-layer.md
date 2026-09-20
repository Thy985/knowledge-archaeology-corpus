# 03 Knowledge Layer — Mem0（Generalized KO）

> 每个 KO 声明 aggregation_rule（R1-R4）+ 簇内 EK + 解释范围扩大论证。层级 L3（Pattern）/L4（Cognitive Model）/L5（Methodology）。

## KO-01 记忆写入"压缩-去重-落库"三段式（L3）
- **aggregation_rule**: R2 因果链簇（EK-01→EK-02→EK-06→EK-09）
- **内容**: 外部记忆系统的写入路径 = LLM 压缩对话为自包含事实（单遍）→ 确定性去重（hash/向量）→ 双写落库（向量 + 事件历史）。压缩在前、去重在压缩后、落库有兜底。
- **解释范围扩大**: 从 Mem0 的 add 管道推广到任何"对话→记忆"系统的通用骨架；letta-code 的 agent 编辑记忆与 hermes-agent 的 MEMORY.md 快照都可映射到该三段式。
- **证据**: main.py:879-1170（管道）；EK-01/02/06/09

## KO-02 混合检索三信号互补（L4）
- **aggregation_rule**: R1 机制簇（EK-09↔EK-10↔EK-11↔EK-13，跨 semantic/keyword/entity 三个独立子系统）
- **内容**: 语义检索（向量）为主信号、关键词（BM25）补词面、实体关系补精确面；三者加性融合且语义阈值前置门控——**召回不是单选，是互补信号的有序融合**。
- **解释范围扩大**: RAG/记忆/搜索系统的通用设计维度——单一检索器总有盲区（向量漏精确词、BM25 漏语义），混合是常态而非特例。
- **证据**: scoring.py:60-130；main.py:1628-1727；EK-09/10/11/12/13
- **Epistemic**: Pattern（Mem0 内强证据；跨项目验证 pending——需 RAG/记忆类项目对照，勿误引 KV 缓存类项目如 dynamo）

## KO-03 LLM 输出防御链（"防幻觉前缀"）（L3）
- **aggregation_rule**: R3 不变量簇（EK-02 hash 去重 + EK-05 UUID 映射 + EK-12 阈值门 + EK-29 JSON fallback 汇聚到同一不变量："LLM 输出不可信，进入系统前必须净化"）
- **内容**: 所有 LLM 生成的中间产物（抽取文本、id 引用、JSON）都要过确定性闸门：id 映射（消除幻觉引用）、hash 去重（消除重复）、阈值门（消除弱信号）、解析 fallback（消除格式噪声）。
- **解释范围扩大**: 任何"LLM 输出进存储/执行"的管道（抽取、分类、规划）都应插确定性闸门。
- **证据**: main.py:873-878、920-945；scoring.py:96-99；test_chatty_llm_parsing.py

## KO-04 OSS 漏斗产品化（L4）
- **aggregation_rule**: R4 主题簇（EK-21 telemetry + EK-22 notices + EK-23 功能桩 + EK-24 匿名身份 + EK-25 迁移信号——同属"开源 SDK 的商业化基础设施"主题）
- **内容**: 开源 SDK 不只是软件，是**获客漏斗的顶层**：遥测采样（成本可控的信号）、远程配置提示（不发版的行为引导）、平台专属功能桩（价值主张展示）、匿名身份别名合并（用户旅程连续）——OSS 的每个机制都在为平台转化服务。
- **解释范围扩大**: 商业化开源（OSS-powered startup）的通用架构模式——需平衡开发者信任（遥测透明度）与转化目标。
- **证据**: telemetry.py:30-72；notices.py；client/main.py:219-245；main.py:515-545
- **Epistemic**: Cognitive Model（Mem0 内强证据，跨项目对照：需更多 OSS 商业化项目验证——见 Candidate C-02）

## KO-05 适配器工厂 + 配置分层（L3）
- **aggregation_rule**: R4 主题簇（EK-31 异常层次 + factory.py provider→class 映射 + 53 配置类 + EK-30 能力检测降级）
- **内容**: 多供应商 SDK 用"ABC 基类 + 工厂映射表 + provider 配置类"三层装配；运行期能力检测（keyword_search）显式降级而不是文档假设。
- **解释范围扩大**: 通用"多后端抽象"模式（类似 SQLAlchemy dialects / LangChain providers）——能力差异必须在运行期暴露。
- **证据**: utils/factory.py；mem0/vector_stores/base.py；main.py:548-556

## KO-06 诚实失败优先（L3）
- **aggregation_rule**: R3 不变量簇（EK-04 LLM 重抛 + EK-20 存储错误 5xx 映射 + EK-31 异常层次汇聚到"错误面必须可区分"）
- **内容**: 错误语义必须诚实：LLM 不可用 ≠ 无事实可抽；存储故障 ≠ 请求错误；功能不支持 = 显式 raise（timestamp）而非静默忽略。**可重试错误与正常空结果必须可区分，否则调用方无法决策。**
- **解释范围扩大**: 库/API 设计的通用原则——错误面映射（4xx vs 5xx）与空结果语义是调用方重试逻辑的输入。
- **证据**: main.py:896-903、2040-2050；exceptions.py

## KO-07 事件溯源记忆历史（L3）
- **aggregation_rule**: R2 因果链簇（EK-14 history 表 + EK-17 不可变身份 + EK-35 双时间戳 + EK-18 实体清理——"每次变更都留痕"链）
- **内容**: 记忆系统对每次变更（ADD/UPDATE/DELETE）写不可变事件（含 old/new 值 + is_deleted 标记 + actor/role），供审计与历史还原；更新不改 created_at。
- **解释范围扩大**: 数据类系统的审计底座通用模式（事件溯源/变更日志），记忆系统尤其需要（用户偏好变更历史有可解释性价值）。
- **证据**: storage.py:150-193；main.py:2082-2126

## KO-08 开源治理双门（L3）
- **aggregation_rule**: R4 主题簇（EK-34 AGENTS.md 双门 + server 认证治理 EK-26/27/28——"供应链与运行时双向治理"主题）
- **内容**: 高热度开源仓库用流程机器人（bot）强制贡献门：CLA + accepted issue 链接 + workflow 凭据 pin；运行时（自托管 server）用 JWT/API key/限流/脱敏治理。**贡献面与运行面分开治理。**
- **解释范围扩大**: 供应侧安全（恶意 PR/依赖投毒）与运行侧安全是两套系统，前者靠流程+bot，后者靠认证+审计。
- **证据**: AGENTS.md；server/auth.py

## KO-09 记忆 scope 隔离契约（L3）
- **aggregation_rule**: R3 不变量簇（EK-17 身份键不可变 + EK-15 拒绝顶层实体参数 + EK-16 提升键 + EK-D1 session_scope——"记忆归属必须显式且不可变"）
- **内容**: 记忆按 user_id/agent_id/run_id 三层 scope 隔离；scope 是创建时的契约（更新不可改）；检索必须显式声明 scope（否则 ValueError）；scope 键在结果中提升为顶层。
- **解释范围扩大**: 多租户/多 agent 记忆隔离的通用设计——归属决定可见性，归属不可变。
- **证据**: main.py:1438-1441、165-260；EK-17/15/16

## 层分布
L3 × 7（KO-01/03/05/06/07/08/09）｜ L4 × 2（KO-02/04）｜ L5 × 0（无跨项目充分证据形成方法论）

## 聚合规则覆盖率
9/9 KO 声明 aggregation_rule（R1×1 / R2×2 / R3×3 / R4×3）；无"同子系统=聚合理由"的假聚合。
