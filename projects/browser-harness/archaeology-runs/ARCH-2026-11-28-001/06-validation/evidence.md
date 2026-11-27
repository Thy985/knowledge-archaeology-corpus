# Evidence Register — browser-harness 0.1.15

> Repository is the source of truth. Each row is a checked anchor; a single-source row is an Observation, not a Fact.
> Code anchors are v0.1.15 `373aecc4`; test anchors are file:line in the same tree.

| EV | Anchor | Claim supported | Source count |
|---|---|---|---|
| EV-01 | `run.py:276-286`, `run.py:472-474` | stdin script is read and exec'd | 2 |
| EV-02 | `run.py:202-245`, `:248-260` | trace max 500, no args in trace | 2 |
| EV-03 | `helpers.py:828-844` | agent_helpers injected into globals | 2 |
| EV-04 | `daemon.py:1,503-522,1276-1300` | daemon holds CDP and serves | 3 |
| EV-05 | `helpers.py:41,75-88`, `_ipc.py:130-150` | per-call IPC request | 2 |
| EV-06 | `_ipc.py:47`, `admin.py:422-446` | protocol 2 compatibility | 2 |
| EV-07 | `daemon.py:1,503-505`, `_ipc.py:68-78`, `run.py:25-47` | one daemon per name; local alias | 4 |
| EV-08 | `daemon.py:1350-1360` | no CDP reconnect | 1 code + test implicit |
| EV-09 | `_ipc.py:34-39` | owner env fallback | 1 |
| EV-10 | `helpers.py:109-118`, `_ipc.py:36-38` | subagent inherits parent | 2 |
| EV-11 | `helpers.py:114-118,441-445,521-525` | refusal only two helpers | 3 |
| EV-12 | `daemon.py:503-522,608-617,619-633` | one owner per tab | 3 |
| EV-13 | `daemon.py:487-501` | Owner fields | 1 |
| EV-14 | `daemon.py:37,1210-1219` | guarded Target methods | 2 |
| EV-15 | `daemon.py:560-576,1216-1219` | iframe proof | 2 |
| EV-16 | `daemon.py:548-556`; `test_daemon.py:533-556` | visibility blanking | 2 |
| EV-17 | `daemon.py:817-831,1219-1223` | session routing | 2 |
| EV-18 | `daemon.py:1219-1245` | no-session pinned to current | 1 code |
| EV-19 | `daemon.py:1226-1228,1219-1223,503-522` | browser session not owner-bound | 2 code paths |
| EV-20 | `daemon.py:662-671`; `test_daemon.py:305-325` | fresh background tab | 2 |
| EV-21 | `daemon.py:666-671` | cloud spare adoption | 1 |
| EV-22 | `daemon.py:1192-1197`, `helpers.py:510-518` | new_tab reuse | 2 |
| EV-23 | `daemon.py:847-855` | popup inherits opener | 1 |
| EV-24 | `daemon.py:726-750`; `test_daemon.py:970-987` | idle keeps current, closes nothing | 2 |
| EV-25 | `daemon.py:752-766`; `test_daemon.py:1071-1085` | forget last tab | 2 |
| EV-26 | `daemon.py:726-750`; `test_daemon.py:1016-1028,1182-1193` | release closes only created unmarked, not last | 3 |
| EV-27 | `daemon.py:151,1124-1134`; `helpers.py:455-460` | marks and persistence | 3 |
| EV-28 | `daemon.py:1136-1145`; `test_daemon.py:557-568` | close_tab ownership | 2 |
| EV-29 | `daemon.py:1230-1245`; `test_daemon.py:784-810` | stale session same tab | 2 |
| EV-30 | `daemon.py:156-180,712-724` | title marker, no session ids | 2 |
| EV-31 | `admin.py:391-411`, `_ipc.py:151-170` | ping strict | 2 |
| EV-32 | `_ipc.py:173-210`, `admin.py:904-1040` | identify type/range | 2 |
| EV-33 | `admin.py:279-336,700-720` | spawn lock | 2 |
| EV-34 | `admin.py:422-446` | newer daemon never stopped | 1 |
| EV-35 | `daemon.py:1079-1092`, `admin.py:422-446` | active owners block | 2 |
| EV-36 | `admin.py:587-620`, `daemon.py:286-300` | cdp-mismatch | 2 |
| EV-37 | `_ipc.py:80-105`, `admin.py:760-790` | remote marker fail closed | 2 |
| EV-38 | `admin.py:192-278,904-1040` | pending fingerprint | 2 |
| EV-39 | `_ipc.py:246-262`, `paths.py:19-23` | POSIX 0600 dir/socket | 2 |
| EV-40 | `_ipc.py:263-278`, `daemon.py:1080-1086` | Windows token | 2 |
| EV-41 | `_ipc.py:39-44,212-230` | 32 MiB cap | 2 |
| EV-42 | `daemon.py:694-710`; `test_daemon.py:1205-1219` | current tab only Network | 2 |
| EV-43 | `daemon.py:826-838` | download owner routing | 1 |
| EV-44 | `daemon.py:853-858` | tab-crashed typed | 1 |
| EV-45 | `daemon.py:619-633,847-852` | destroy drops ownership | 2 |
| EV-46 | `daemon.py:867-909,912-1000` | captcha per-owner tab | 2 |
| EV-47 | `daemon.py:635-660`, `_ipc.py:73-75` | owners JSON persistence | 2 |
| EV-48 | `daemon.py:645-660` | damaged state tolerated | 1 |
| EV-49 | `daemon.py:164-176,752-770` | idle exit policy | 2 |
| EV-50 | `daemon.py:1350-1360` | CDP death stops daemon | 1 |
| EV-51 | `daemon.py:1301-1341` | owner PID watch | 1 |
| EV-52 | `daemon.py:1362-1378,1094-1106` | signals normal shutdown | 2 |
| EV-53 | `report.py:45-160` | report scrub | 1 module, multiple regexes |
| EV-54 | `telemetry.py:22-40,149-163` | property key filter | 2 |
| EV-55 | `helpers.py:771-820` | private HTTP stays local | 1 |
| EV-56 | `SKILL.md:52`; `daemon.py:1220-1228,1219-1223,503-522` | raw CDP availability vs browser-level gap; README quote absent | 4 |
| EV-57 | `AGENTS.md`; `SKILL.md:2`; `pyproject.toml` | package/skill identity split | 3 |
| EV-59 | `run.py:207-237,323-356`, `telemetry.py:282-315`, `helpers.py:843`; 0.1.14 wheel diff | cli_event trim, no 0.1.15 changelog | 4 |
| EV-60 | `daemon.py:635-643`, `_ipc.py:73-75`, `paths.py:19-23` | owners JSON mode not explicitly tightened | 3 |
| EV-58 | `browser-use` 1.0.8 `pyproject.toml:19`, `browser_use/browser/browser.py:900-940`, source-wide grep | pin 0.1.13, no owner API, unique daemon per non-shared browser | 3 |

## Single-source / unverified rows

- EV-08, EV-09, EV-13, EV-18, EV-21, EV-23, EV-34, EV-43, EV-44, EV-48, EV-50, EV-51, EV-53, EV-55 are single-code-source anchors. They are Observations, not Facts, unless a test or second module corroborates.
- EV-58's pin and grep are Facts; the concurrency consequence is an inference and is marked as Hypothesis in C-08.
- No test command was executed in this run. Test files are evidence of intended behavior, not proof that the suite passes at this commit.
