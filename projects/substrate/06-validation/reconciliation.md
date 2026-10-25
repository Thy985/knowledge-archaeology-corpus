# Reconciliation — ARCH-2026-10-26-001

> 主考古产物（00-06 + run_metadata）与 Independent Validation（盲重建）冲突的显式合并记录。
> 原则：修正 Corpus Artifact，不改生产 Skill；不弱化证据标准。

## 修正项（5）

| # | 对象 | 类型 | 修正内容 | 依据（盲重建证据） |
|---|---|---|---|---|
| R-1 | 04_flow_atlas Edge C3b（新增） | 补遗漏 | worker pod port 80 在 `--upstream-connect=false` 且无 credential bundle 时是可达 legacy plaintext 路径；"80 非直接入口"仅默认配置成立 | cmd/atenet/internal/router/cmd.go:77 注释（"disables upstream mTLS (legacy plaintext pod-IP:80)…needs --upstream-connect=false"） |
| R-2 | 04_flow_atlas Control Flow | 补遗漏 | router 数据平面端口维度：--port-https=8443 / --port-connect-tls=8444（与 atunnel 侧 8443=CONNECT / 443=HTTPS 不同） | cmd.go:63,70 |
| R-3 | 02 EK-S10 | 证据升级 | "凭据注入"证据级 [doc]→[code]+[doc]（注入路径为代码事实；"丢弃后不可读"子句仍为文档契约） | cmd/atenet/internal/router/egress/credentials.go（applyEffects/providerCredential/actorJWT） |
| R-4 | 01_project_map | 精度 | EgressPolicy 作用域细化：actor 级（Get/Create/Update/DeleteActorEgressPolicy）+ Global/Atespace AccessPolicy 并存 | api/ateapi/v1/ateapi.proto |
| R-5 | run_metadata.yaml | 元数据 | independent_validation 判定统计更新（CONFIRMED 11/2 PARTIAL/1 CONTRADICTED/4 MISSING/1 REVIEW/0 DOWNGRADED） | 本 reconciliation |

## 未合并（保留差异）
- C-S05（凭据外注三仓收敛）标 Cross-project validation pending——不因"结论漂亮"升 Principle（保持 Hypothesis）
- security.md "Met" 声明群（Dual-Party Snapshot Authorization 等）保留 NEEDS_HUMAN_REVIEW——不替人工下结论

## Gate 检查（写入前）
- 文件数：projects/substrate/ = 14（README/metadata/00/6 目录 9 文件/06-validation×3/archaeology-runs×4）
- schema：metadata.yaml 字段与 thinkingbox 模板一致；目录名与 delivery.md 约定一致
- duplicate：无重复 run 目录（ARCH-2026-10-26-001 唯一）；无与已有 36+ 项目 slug 冲突
- evidence-ref：EK/KO/Flow 证据锚均为本 run 快照内文件（repo commit 59a58a5f）
- metadata：commit/run_id/skill_version/日期齐全
- structure：01-06 六层 + archaeology-runs 布局与模板一致
