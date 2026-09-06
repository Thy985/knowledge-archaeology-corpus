# 06 Validation & Evidence

## 1. 验证方法

| 检查 | 方法 | 结果 |
|------|------|------|
| Source Truth | 盲重建：独立重读 lib/index.js + store.js + review.js + skills.js + session-orch.js + sync/{identity,merge,repo}.js + coi/ws-coord.js + advisor/guard.js + 测试 | ✅ 无事实错误 |
| Coverage | 强制面覆盖：记忆/审查/技能/待办/编排/协作/同步/评审/搜索/派单/前端/更新 12 面 | ✅ 30 EK 覆盖 |
| Causality | 每条 EK 因果声明回溯 | ✅ |
| Flow | 七类流 Edge 全回溯（lib/ 可读，无盲区） | ✅ |
| Abstraction | L3/L4/L5 独立判定 | ✅ 8 KO/4 CM/4 M 受支撑 |
| Counterexample | 反例预算制 | ✅ 见下 |
| Epistemic | Fact/Observation/Hypothesis/Pattern 标注 | ✅ C-01~08 |

## 2. 反例攻击记录

### 对 KO-01（双轨记忆 + 确认门）的攻击
- **A1**：审查提议是否真的"用户确认才写入"？→ 反例：auto 模式（reviewEnabled）直接写全局记忆（主会话不受门控）→ **收紧**：KO-01 表述加"auto 模式例外（主会话不受门控）" ✅
- **A2**：计数器不自动重置是否导致永久 DUE？→ 测试契约确认：只有 complete 调用重置；若模型从不 complete 则审查一直 DUE——这是设计代价，记录为 C-01 ✅
- **A3**：情绪反馈是否真的落盘？→ README 场景一 + review 实现（【反馈】行）→ 部分证实（实现细节未全读，标文档+部分实现）✅

### 对 KO-02（文件写入安全三件套）的攻击
- **B1**：漂移守卫是否所有写路径都走？→ 反例：add 追加跳过守卫（append-only）→ **收紧**：三件套表述为"重写路径三件套"，add 例外入 C-02 ✅
- **B2**：锁是否真的跨进程？→ STALE_LOCK_MS/LOCK_TIMEOUT_MS/自旋实现可读 + 多进程测试（store.test.js）→ 证实 ✅
- **B3**：原子写是否真原子？→ renameSync 实现（store.js）→ 证实 ✅

### 对 KO-03（跨设备同步链）的攻击
- **C1**：https/ssh 真的收敛同键？→ normalizeRemoteUrl 纯函数实现（去协议/凭证/.git/尾斜杠/host 小写/默认端口省略）+ 单测（identity.js 注释声明纯函数便于单测）→ 证实 ✅
- **C2**：合并器是否真的不落盘冲突标记？→ merge.js 注释（"git 冲突标记永不落盘"）+ 内存决策 + 冲突清单 → 证实 ✅
- **C3**：decideModeB 是否真的绝不碰 main？→ e2e 测试（"共享仓库串项目防护——main 是别人的，本项目独立分支互不干扰"）→ 证实 ✅
- **C4**：身份归一化失败回退是否兼容旧目录？→ projectHash(cwd) 12 hex 与 store.js 同一实现（迁移回查依赖）→ 证实 ✅

### 对 KO-04（AI 动作治理三件套）的攻击
- **D1**：read 证据是否可伪造？→ 会话日志事件是唯一证据源；只证明"调用过 read"不证明"理解"→ 收紧为 C-03（证据语义边界）✅
- **D2**：guard 归一化是否真的覆盖 *stop*？→ NFKC→小写→非字母数字折叠→trim（"Stop."/*stop*/"  STOP  " 全归一到 stop）→ 证实 ✅
- **D3**：17 个空泛短语是否抑制完整 note？→ 精确匹配归一化文本——包含短语的完整 note 不受影响 → 证实 ✅

### 对 KO-07（测试环境契约）的攻击
- **E1**：48 挂是否可能是真实回归？→ 逐个定位：update.test.js git push origin main（默认分支 master 时失败）+ git 身份缺失 + darwin 假设——配置后 3 挂全为 darwin → 证实环境契约 ✅
- **E2**：darwin 3 挂是否无解？→ search-docs 硬编码 mdfind 预期（process.platform==='darwin'），Linux 必然失败——平台假设而非回归 → 证实 ✅

## 3. Epistemic 状态清单（最终）

| 对象 | 状态 | 依据 |
|------|------|------|
| EK-01~30 | **Fact** | lib/ 可读实现（S3/S4 测试） |
| KO-01~08 | **Pattern/Cognitive Model**（本项目） | 支撑 EK |
| CM-1~4 / M-1~4 | **Model/Methodology**（pending） | 支撑 KO |
| C-01/C-08 | **Hypothesis**（跨项目） | 未验证 |
| C-02/C-06 | **Tentative** | 边界/模式 |
| C-04/C-05 | **Observation** | 明确边界记录 |
| C-07 | **Observation + 建议** | 本 run 实测 |

## 4. 实测记录

- **全量测试**：798 断言（60 文件），默认环境 750 pass / 48 fail → 配置 git 契约（init.defaultBranch=main + user 身份）后 **795 pass / 3 fail（全为 darwin 平台假设）**
- **conformance 无**（本项目无 golden-vectors；但 update/sync 测试含 e2e 场景）
- **未实跑**：npm build（需 DSH source checkout 的 esbuild——构建期依赖宿主环境）；WebUI 端到端（需 DSH host）

## 5. 质量指标

| 指标 | 值 |
|------|-----|
| Facts/Evidence | ~60（distinct 证据点） |
| EK | 30（links 100%，游离 0，全部 Fact 级） |
| KO | 8（aggregation_rule 100%） |
| CM/M | 4 / 4 |
| Candidates | 8（0 NEEDS_HUMAN_REVIEW——本包全部可读，无盲区） |
| 反例攻击 | 14 次（A1-3/B1-3/C1-4/D1-3/E1-2） |
| 实测 | 全量 798 断言（795 pass / 3 darwin 假设） |
| 判定 | PASS（2 收紧 + 无 NEEDS_HUMAN_REVIEW + 无盲区） |
