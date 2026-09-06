# 05 Candidates — 未确认内容

## C-01 [Cross-project Hypothesis] GPL-3.0-or-later 声称 vs 混淆源码的合规边界
- **内容**：仓库声明 GPL-3.0-or-later 但 57/202 src 文件混淆（evolve 全模块 + gep 半数）。GPL 要求提供对应源码（corresponding source）——混淆源码是否构成"对应源码"是法律灰色区；README 已公告"转向 source-available"（未来版本不再 fully open source）。Hypothesis：混淆动机与 2026-03 Hermes Agent 相似性争议（README Notice 声明）直接相关。
- **证据**：README Notice（L20-30）；grep _0x 统计；package.json devDeps
- **epistemic**: Fact（混淆与声明并列）/ Hypothesis（动机与合规影响）
- **scope**: Cross-project（影响所有评估"开源"AI 工具的人）

## C-02 [Tentative Finding] doc-impl 矛盾：validation 白名单 npm/npx
- **内容**：README Security Model 称 validation 命令前缀白名单 node/npm/npx；实现 ALLOWED_EXECUTABLES=['node'] only（npm/npx 已因 GHSA-jxh8-jh77-xh6g 移除）。文档未同步安全收紧——**文档比实现"更宽松"**（与 skillfortify C-01 的"文档更激进"方向相反，但同属 doc-impl 分裂）。
- **证据**：README L494-499 vs sandboxExecutor.js L44 vs test 断言
- **epistemic**: Fact（矛盾本身）；需要人类裁决：README 是过时还是实现过严

## C-03 [Observation] "prompt generator, not code patcher" vs solidify 修改 src/**
- **内容**：README 称 Evolver 不自动编辑源码；但 solidify 存在（可修改 src/**——SKILL.md file_access "src/** evolved code, only during solidify"）+ EVOLVE_ALLOW_SELF_MODIFY 开关存在（默认 false）。语义需澄清：solidify 是"生成补丁供宿主应用"还是"直接应用补丁"（混淆实现不可读）。
- **证据**：README L196-208 vs SKILL.md file_access vs EVOLVE_ALLOW_SELF_MODIFY
- **epistemic**: Observation；**NEEDS_HUMAN_REVIEW**（涉及"自进化引擎到底改不改自己的码"的核心语义）

## C-04 [Tentative Pattern] 源码级回归测试（测试即静态分析）
- **内容**：fetchSecurity 直接 grep index.js 源码模式（outFlag.slice/path.resolve/rel.startsWith('..')）锁定 GHSA 修复。此模式的前提是**源码可读**——对混淆项目失效（evolve 模块无法用此方法回归）。是否值得推广为通用模式（结合 AST/解析器）待验证。
- **证据**：test/fetchSecurity.test.js（全文模式）
- **epistemic**: Pattern（本项目）/ Cross-project pending

## C-05 [Scope-uncertain] 自我修改的"最后一行"边界
- **内容**：EVOLVE_ALLOW_SELF_MODIFY=false 默认 + 回滚保护存在，但"进化生成的补丁最终由谁应用"（宿主？evolver 自己？sandboxExecutor 内？）在可读面不完整——混淆的 solidify 实现是唯一权威源。
- **证据**：SKILL.md 注释 + test 契约 + 混淆 solidify.js🔒
- **epistemic**: Hypothesis（边界不确定）

## C-06 [Potential Finding] conformance golden-vectors 的扩展价值
- **内容**：savings-core conformance（spec_version 0.3.0）用 golden-vectors 锁 token 节省公式（62.61% 精确匹配）。该"公式级一致性基准"模式对任何"声称收益可量化"的工具（RAG 节省/进化收益）有普适价值。
- **证据**：conformance/savings-core/golden-vectors.json；实测 PASS
- **epistemic**: Observation（本项目）/ Cross-project 价值待验证

## C-07 [Tentative Finding] ATP 协议 SDK 化的防漂移设计
- **内容**：ATP 枚举从 @evomap/atp-sdk 单一来源导入（注释明示 v1.80.8 explore enum 事故教训）。"协议枚举进独立 SDK 防止多运行时漂移"是否成为协议治理通用模式待验证。
- **证据**：src/atp/protocol.js 全文
- **epistemic**: Pattern（本项目）/ Cross-project pending

## C-08 [Observation] skill 系统自指生态（skill → GEP 资产转化）
- **内容**：Evolver 自带 SKILL.md（capability-evolver 自我描述）+ skill2gep/skillDistiller/skillPublisher 把 Claude 风格 skill 转化为 GEP 基因——"skill 协议"与"GEP 协议"的桥接层。与 knowledge-archaeology skill 的 SKILL.md 同为自我描述 skill；跨项目比较价值待验证。
- **证据**：SKILL.md 全文；scripts/skill2gep.js；src/gep/skillDistiller.js
- **epistemic**: Observation

---

## 候选处置

| 候选 | 处置 |
|------|------|
| C-01 | 保持 Hypothesis（混淆动机/合规影响），跨项目 |
| C-02 | 保持 Fact（矛盾确凿）→ NEEDS_HUMAN_REVIEW（README 过时 or 实现过严） |
| C-03 | 保持 Observation → NEEDS_HUMAN_REVIEW（solidify 应用边界） |
| C-04/C-07 | 保持 Tentative Pattern |
| C-05 | 保持 Hypothesis |
| C-06/C-08 | 保持 Observation |
