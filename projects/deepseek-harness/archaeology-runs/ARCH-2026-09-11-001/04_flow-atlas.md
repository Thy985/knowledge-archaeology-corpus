# 04 — Flow Atlas（v0.1.5 增量）

> 规则：从真实代码/notes 导出，关键 Edge 必须可回溯 symbol/file/condition/state transition。基线 7 类 Flow 见 09-03 考古包 04_flow-atlas/。本节补充 v0.1.5 新增/修正的边。

## 4.1 Control Flow（新增：PTC 执行路径）

```
模型 tool-call (native name, nested=false)
  → ToolRuntime.resolveExecution(name, scope, nested)      [packages/*/tool-runtime]
  → collapses(name, nested)?  [modeFor(scope)==ptc && !nested && name!='run_code']
      ├─ YES → undefined → UNKNOWN_TOOL（错误消息回指 run_code）
      │         └─ ABORTED_BEFORE_DISPATCH（已中止调用信号，visible tool finalizer 应用）
      └─ NO  → createExecution（prepare 第一阶段）
                → pre-execute 监听 → approval ask → guards → execute
run_code SDK 绑定 (nested=true, parent token 已设)
  → resolveExecution 放行全部可见工具（SDK 声明过的 binding 可用）
```
**关键 Edge**：collapse 判定在 `createExecution` **之前**——pre-execute/approval/guards **永不观察**确定性拒绝的调用；人类永不被打扰审批。非 JSON 可序列化参数 → TypeError（invalid-args 契约）而非 UNKNOWN_TOOL。

## 4.2 State Flow（新增：V3 Session envelope）

```
事件写入:  seed/append/restoration
  → Core Session surface.ts 校验（event-local placement / header 空字段 / tool-error 规则）
  → 关系校验（surface manager，需事件日志：成员资格/有序端点/引用覆盖/内容单节点替换）
  → 接受 → 追加到 append-only JSONL（surfaceOp=append）
  → 替换路径: {op:'replace', startSeq, endSeq}（端点=当前 surface 序，inclusive，先于被替换事件）
tool/result with data.error → message.content[0].isError === true（强制）
矛盾元数据 → 不推断错误结局（当前读取与迁移均不推断）
```

## 4.3 Data Flow（新增：workspace-files）

```
Client file provider → workspaceFiles Remote namespace（wire Session identity）
  → Gateway: live Session header 或 SessionPersistence.stat（cold）→ WorkspaceFileScope{id, cwd}
     [不激活 Agent / 不读事件体 / 不回退父 Session]
  → Host 方法（read/readBytes/readAll/readRelated/stat/list/changes）
     → FileSystem.readByteRange(target, signal, maxBytes)   [dsh-fs seam，所有 provider 实现]
     → 结果按绝对路径命名（execution world 绝对路径）
     → 内容 cap：page / byte-window / complete-file
list/changes → workspace 收敛（后端 workspace-containment 谓词）
```

## 4.4 Authority Flow（新增：读权限分界）

```
read/readBytes/readAll/readRelated/stat
  → 继承 Session fs 后端读授权（workspace 根 = 相对路径基准，非读边界）
  → 绝对路径 / '..' 越界：后端允许即可读（readRelated 从基文件目录解析）
list / changes
  → workspace-scoped（导航/观察 ≠ 具名读取）
Document Preview
  → 打包静态声明 JS/CSS → HTML Blob iframe sandbox="allow-scripts"（opaque origin）
  → 父应用不可访问；浏览器网络保留（intentional trade-off）
```

## 4.5 Evidence Flow（增量）

- python SDK 测试 = **S4 实测证据**（111 passed / 7 skipped，`python/sdk/tests`）。
- notes 体系 = 决策证据源（EK-R01）：Problem→Decision→Alternatives 结构，双语配对，frozen archive。

## 4.6 Memory / Policy Flow

- 记忆流：无变化（09-03 KO-04 四层上下文治理成立）。
- Policy Flow：新增闭环实例——09-05 决策（containment）→ 09-09 supersede（继承 fs 授权）→ 记录 Alternatives → 未来决策引用（EK-R05）；PTC collapse 使 policy 管线可见性收窄（策略只处理未折叠调用）。

## 4.7 Flow→KO 交叉校验

| KO | 校验的 Flow Edge | 结果 |
|---|---|---|
| KO-09 | 4.1 中 collapse 先于 createExecution 的 edge | 通过（真实 symbol resolveExecution/collapses） |
| KO-10 | 4.4 中 read vs list 的不同边界 edge | 通过（read 族 vs list/changes 真实分界） |
