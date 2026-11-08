# EK Graph

## 1. 边表（25 条）

| # | 类型 | From | To | 语义 |
|---|---|---|---|---|
| 1 | subsystem | EK-01 | EK-02 | 同一记忆子系统 |
| 2 | dependency | EK-01 | EK-07 | Core 写入依赖服务端 diff 路径 |
| 3 | mechanism | EK-02 | EK-16 | 准入机制在 Recall 上表现为压缩 |
| 4 | subsystem | EK-02 | EK-14 | Archival 是三级之一 |
| 5 | subsystem | EK-03 | EK-19 | 共享载体导致权限问题 |
| 6 | mechanism | EK-04 | EK-05 | 两种版本化机制 |
| 7 | constraint | EK-20 | EK-04 | 删除约束版本化能力 |
| 8 | contrast | EK-05 | EK-04 | 同一问题的两代解法（DB 快照 vs git） |
| 9 | dependency | EK-06 | EK-05 | 补偿机制依赖 git-as-truth 声明 |
| 10 | causal | EK-07 | EK-08 | diff 路径触发字符串比较 |
| 11 | causal | EK-07 | EK-09 | 写入路径经过 executor 门禁 |
| 12 | dependency | EK-08 | EK-09 | 写判定依赖门禁后的内存状态 |
| 13 | contrast | EK-09 | EK-10 | 同一约束的两种实现层 |
| 14 | mechanism | EK-10 | EK-09 | 都是 read_only 约束的实现 |
| 15 | constraint | EK-11 | EK-09 | schema 直调被禁，语义落到 executor |
| 16 | mechanism | EK-12 | EK-13 | 都是精确编辑机制 |
| 17 | constraint | EK-13 | EK-11 | 编辑约束由真实实现承担 |
| 18 | subsystem | EK-14 | EK-02 | archival 写入 |
| 19 | causal | EK-15 | EK-14 | 双写失败由写入路径产生 |
| 20 | mechanism | EK-16 | EK-17 | 压缩与"非原文回放"同属 Recall |
| 21 | subsystem | EK-16 | EK-02 | Recall 压缩 |
| 22 | contrast | EK-17 | EK-01 | 与 Core 常驻形成对照 |
| 23 | constraint | EK-18 | EK-19 | 租户隔离约束共享权限设计 |
| 24 | subsystem | EK-19 | EK-03 | 权限缺口源于共享 block |
| 25 | contrast | EK-19 | EK-18 | 块级权限 vs 组织级隔离 |
| 26 | constraint | EK-21 | EK-04 | 重复定义影响历史回放实现 |
| 27 | causal | EK-22 | EK-06 | 字段同步遗漏加剧缓存不一致 |
| 28 | dependency | EK-22 | EK-05 | 依赖 git-as-truth 的缓存假设 |
| 29 | mechanism | EK-23 | EK-10 | 同属 legacy 无校验路径 |
| 30 | contrast | EK-23 | EK-11 | legacy 可创建 vs v3 仅编辑 |

## 2. 统计

- EK 节点：**23**（审计后新增 EK-21/22/23）
- 边：**30**
- 平均出边：30 / 23 = **1.30**
- 游离节点：**0**
- 边类型分布：subsystem 5、mechanism 5、dependency 4、causal 4、constraint 5、contrast 5、其他 2

## 3. 图结构特征

```
                 EK-02 (三大准入)
                ╱      ╲
         EK-16(压缩)   EK-14(archival写)
           │             │
         EK-17        EK-15(双写失败)
           │
        (contrast) EK-01(Memory=Core)
                     │
                   EK-07(写路径)
                   ╱        ╲
              EK-08(字符串diff) EK-09(read_only门禁)
                                  │     ╲
                              EK-11    EK-10(legacy无校验)
                                  │
                              EK-13(精确编辑)

        EK-03(共享block) ── EK-19(权限缺口) ── EK-18(租户隔离)
                                  │
                              EK-04(BlockHistory) ── EK-05(git truth) ── EK-06(无事务)
                                  │
                              EK-20(删除级联)
```

## 4. 供 KO 聚合的簇

| 簇 | 规则 | 成员 | 候选 KO |
|---|---|---|---|
| 三级准入 | R1 机制簇 | EK-01/02/14/16/17 | KO-01 |
| 记忆版本化 | R2 因果链簇 | EK-04→EK-05→EK-06（+EK-20 约束） | KO-02 |
| 写入门禁 | R3 不变量簇 | EK-07/08/09/10/11/13 | KO-03 |
| 共享与授权 | R3 不变量簇 | EK-03/18/19 | KO-04 |
| 记忆删除 | R2 短链 | EK-20→EK-04 | 并入 KO-02 的边界 |
| 变更检测 | R4 主题簇 | EK-08（单点，归入 KO-03） | — |
