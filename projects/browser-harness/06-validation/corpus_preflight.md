# Corpus preflight checks — ARCH-2026-11-28-001

| check | result |
|---|---|
| structure | PASS: metadata / 00 / 01–06 / run dir present |
| YAML parse | PASS: metadata.yaml, run_metadata.yaml, run dir run_metadata.yaml |
| EK schema | PASS: 60 unique EK IDs, 150 links, 0 orphan, all link targets exist |
| KO aggregation | PASS: 10 KO, each `derivation.facts` equals `cluster_eks`; 0 unknown EK |
| KO level counts | PASS after reconciliation: 9 L3 + 1 L4 + 0 L5 |
| Evidence rows | PASS: 60 unique EV IDs; EK→EV mapping present |
| Candidate IDs | PASS: 9 unique C IDs |
| Duplicate check | PASS: no duplicate EK/KO/EV; new run directory does not exist on `origin/main` |
| History preservation | PASS: old top-level 00–06 archived byte-identical under `archaeology-runs/ARCH-2026-10-04-001/package-snapshot/` |
| Metadata accuracy | PASS: version/commit/tag and run id verified against the target repo |
| Target anchors | PASS for code anchors; one document quote failed and was removed by reconciliation (EK-56) |
| Tests executed | NO (declared) |
| Runtime probes | NO (declared) |
