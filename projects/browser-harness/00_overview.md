# 00 Overview — browser-harness 0.1.15 refresh

- **Project**: `browser-harness` (`https://github.com/browser-use/browser-harness.git`)
- **Run**: `ARCH-2026-11-28-001`
- **Job**: `KA-2026-11-28-001` (priority A, mode `refresh`)
- **Baseline**: `ARCH-2026-10-04-001`, version `0.1.13 @ afbcc381b963040c19627d788e40c7e7663171ee`
- **Target**: tag `v0.1.15 @ 373aecc4c40ef67d97b5ee69c144684a64a42a11` (2026-11-25)
- **Skill**: knowledge-archaeology v3.2
- **Scope**: 299 tracked files; 8,428 source Python lines + 6,830 test lines; 404 `def test_`; shallow clone. No install or test execution in this run.
- **Platform fact used below**: `v0.1.13...v0.1.15` compare on GitHub reports 403 commits ahead. This run reads the v0.1.15 tree, not the intermediate history.

## Headline

0.1.15 turns the previously documented "many agents, one Chrome" idea into a real daemon-level ownership model: a tab has at most one owner, raw `Target.attachToTarget`/`closeTarget`/`activateTarget` are guarded, another owner's tab is URL/title-blanked, idle tabs are reclaimed, and owner→tab state survives daemon restarts. The old C-02 candidate ("multi-agent shared browser lane has no mutex") is therefore **partially resolved**: tab-level ownership is implemented and unit-tested, not merely claimed.

It is not fully resolved. The ownership check is cooperative, not authenticated: the daemon accepts any `owner` string from any process that can reach the socket. A sub-agent that inherits its parent's session id and lacks `BH_OWNER` is warned, and is refused only for `close_tab`/`release_tabs`; it can still drive the parent's current tab through `cdp()`. Browser-level raw CDP also escapes tab ownership: `Target.attachToBrowserTarget` is unguarded and its session id is not recorded in `session_owner`, so the check `session_owner.get(sid, key) == key` passes for a second owner. A browser-level session can then call browser-domain methods (`Storage.getCookies`, `Network.getAllCookies`, etc.) outside any tab owner. A quoted README sentence "Isolation and privacy hold for raw CDP too" **could not be located in this v0.1.15 snapshot** (README is 137 lines; grep for "raw CDP"/"isolation"/"privacy hold" returns no such claim). The only tree statement is `SKILL.md:52` "Raw CDP is available with `cdp(...)`", which is availability, not an isolation guarantee. The code evidence is therefore: tab-routed CDP is guarded; browser-domain CDP is not owner-bound and has no code-level isolation guarantee.

The second structural finding is the version seam. `browser-use 1.0.8` pins `browser-harness==0.1.13` and never mentions `BH_OWNER`, `release_tabs`, `list_browsers`, `use_browser`, `name_session`, or `mark_tab` in its own source (only the vendored 0.1.13 site-packages contains them). Its concurrency strategy is different: each `Browser('chromium'|'cloud'|profile)` gets a unique daemon name `bu<8hex>` and its own browser; only `Browser('chrome')` uses the shared default daemon, with no owner identity. So 0.1.15's owner model is not consumed by the main downstream; the shared-lane risk is displaced to the browser-use shared-Chrome path, not removed.

## What was produced

| Layer | Content |
|---|---|
| `01-project-layer/` | project map, module boundaries, state, config, governance |
| `02-engineering-knowledge/` | 60 EK nodes with six edge types and file:line evidence |
| `03-knowledge-layer/` | 10 generalized KO (9 L3 + 1 L4) with R1–R4 aggregation rules |
| `04-flow-atlas/` | seven flows (Control/State/Data/Evidence/Authority/Memory/Policy) |
| `05-candidates/` | unresolved and cross-project hypotheses, including downgraded C-02 |
| `06-validation/` | self-validation, blind independent audit, reconciliation |
| `run_metadata.yaml` | run identity and quality metrics |

## Most important finding

**Tab ownership is a routing contract, not a security boundary.** It prevents cooperating agents from stepping on each other's tabs and makes those failures explicit (`tab-taken`, `tab-closed`, `tab-released`, `owner-shared`). It does not establish an authority boundary against a same-user process that can write `BH_OWNER`, nor against a browser-level CDP session. Treat it as collision avoidance plus auditability; do not treat it as isolation between mutually untrusted agents.
