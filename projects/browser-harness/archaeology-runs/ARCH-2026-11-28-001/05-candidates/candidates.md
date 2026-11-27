# 05 Candidates — browser-harness 0.1.15

> Candidates are not validated knowledge. Each has an epistemic status, the missing evidence, and a next probe.
> Cross-project candidates never appear as Principles in 03.

## C-01 · Browser-level CDP session escapes tab ownership (Hypothesis, high structural confidence)

- status: Hypothesis; code path confirmed, exploitability unverified.
- claim: `Target.attachToBrowserTarget` returns a session that is not recorded in `session_owner`. Any owner that knows that session id passes the explicit-session check because `session_owner.get(sid, key)` defaults to the caller's key. Browser-domain methods sent on that session (e.g. `Storage.getCookies`, `Network.getAllCookies`) are not tab-scoped.
- evidence: `daemon.py:1210-1228`; `daemon.py:503-522`; `daemon.py:1219-1223`.
- missing: an executed two-owner probe that attaches a browser session as A, sends the sid to B, and reads a cookie belonging to a third tab. Requires a disposable browser profile and consent.
- why it matters: directly contradicts the README's "Isolation and privacy hold for raw CDP too" if the sid is obtainable by another owner.
- next probe: unit test with a fake CDP that records `Storage.getCookies` as browser-wide; then integration only on a throwaway Chrome profile.

## C-02 · Multi-agent shared browser lane: partially resolved, not closed (Hypothesis, downgraded from "no mutex")

- status: Hypothesis; the old claim "SKILL.md claims serial safety, no mutex in repo" is **downgraded**.
- claim: 0.1.15 implements a tab-level mutex (`tab_owner`, `GUARDED`, `session_owner`, idle takeover), so the old "no implementation at all" framing is false. The remaining gap is authority: the mutex is cooperative, not authenticated; browser-level sessions bypass it (C-01); a sub-agent without `BH_OWNER` can drive the parent tab (C-03).
- evidence: EK-12/14/15/16/17/18/19/24/28/29; tests `test_daemon.py:367-589,699,952,1009`.
- missing: a multi-process adversarial probe; a test where one owner deliberately sets another's `BH_OWNER`.
- next probe: two-process harness with the same OS user; A creates a tab; B sets `BH_OWNER=A` and calls `goto_url`; observe whether B drives A's tab (predicted yes).
- benchmark value: this is the clean case for "routing contract vs security boundary."

## C-03 · Sub-agent without `BH_OWNER` can drive the parent's current tab (Hypothesis, code-confirmed)

- status: Hypothesis; code-confirmed, impact depends on host behavior.
- claim: `_shares_parents_owner()` refuses only `close_tab`/`release_tabs`. A Claude child with `CLAUDE_CODE_CHILD_SESSION=1` and no `BH_OWNER` shares the parent's owner key, and `cdp("Page.navigate")` or `js()` still routes to the parent's current tab. The warning is advisory.
- evidence: `helpers.py:109-118`; `helpers.py:441-445`; `helpers.py:521-525`; `helpers.py:96-107`; `_ipc.py:36-38`.
- missing: a real Claude Code sub-agent run; the host may set `BH_OWNER` itself.
- next probe: spawn a child process with `CLAUDE_CODE_CHILD_SESSION=1`, no `BH_OWNER`, and call `js("document.title")` after the parent opens a tab; see whether it reads the parent's tab.
- why it matters: "sub-agents must have their own BH_OWNER" is a doc rule, not a hard invariant.

## C-04 · `agent_helpers.py` can overwrite core helpers (Observation → Hypothesis)

- status: Hypothesis; code path obvious, no test.
- claim: `_load_agent_helpers` copies every public name into `helpers.globals()`, including names that already exist. A later agent-defined `cdp` or `switch_tab` shadows the core function. `_OWN_NAMES` is snapshotted before injection, so tracing labels the shadow as `custom`, but behavior is changed.
- evidence: `helpers.py:828-844`; `run.py:202-240`.
- missing: a test that defines `def cdp(...)` in agent_helpers and observes shadowing.
- next probe: isolated `BH_AGENT_WORKSPACE` with a shadowing helper; call `cdp("Browser.getVersion")`; expect the agent's function, not the core.
- why it matters: the "runtime capability extension" model is also an unguarded monkey-patch surface.

## C-05 · The owner JSON file is readable by the same user and may leak tab targets (Observation)

- status: Observation; no exploit.
- claim: `bu-<name>.owners.json` stores target ids, current tab, marks, created sets, and session names. It is written under the runtime dir (0700) and `_save` uses tmp+replace, but no explicit `0600` on the file is set in `_save`. Same-user processes can read it; a hostile same-user process can also write it.
- evidence: `daemon.py:635-643`; `_ipc.py:73-75`; `paths.py:19-23`; EK-60.
- missing: file mode check; threat model for same-user processes.
- next probe: `stat` the file after a run; inspect mode.
- why it matters: target ids alone are not cookies, but they are handles; combined with C-01 they may matter.

## C-06 · Cross-project: daemon identity mechanisms are isomorphic (Cross-project Hypothesis)

- status: Cross-project Hypothesis; not validated.
- claim: browser-harness's `ping {pong:true}` + `identify` type/range check + start-time fingerprint resembles deepseek-harness and opencode daemon identity patterns. A shared "process identity before signal" model may exist across agent harnesses.
- evidence: EK-31/32/33/38; old corpus `projects/deepseek-harness` and `projects/opencode` (not re-read this run).
- missing: a side-by-side symbol-level comparison with those corpora.
- next probe: read their daemon/pid modules and compare the three checks.
- why it matters: candidate for a cross-project L4, but currently only one side is verified.

## C-07 · "One Allow per Chrome run" holds only under the default naming policy (Hypothesis)

- status: Hypothesis; code-implied.
- claim: One daemon per browser gives one Chrome connection, but `BH_NAMED_LOCAL_DAEMON=1` or a pre-existing named daemon prevents `run.py` aliasing, so two `BU_NAME`s can create two connections and two Allow prompts. The changelog's "one Allow per Chrome run" is a default-policy statement, not an invariant.
- evidence: `run.py:25-47`; `daemon.py:1`; `admin.py:644-720`.
- missing: a run with two `BU_NAME`s against one Chrome and a count of prompts.
- next probe: launch a disposable Chrome, run two named daemons with `BH_NAMED_LOCAL_DAEMON=1`, count `handshake-wait` prompts.
- why it matters: operators may read the claim as a guarantee and be surprised by multiple consent sheets.

## C-08 · Version seam: 0.1.15 owner model is not active in browser-use 1.0.8 (Hypothesis, high confidence on facts)

- status: Hypothesis; pin and API absence are facts, user impact is inferred.
- claim: browser-use 1.0.8 pins `browser-harness==0.1.13`; its source has no `BH_OWNER`/`release_tabs`/`list_browsers`/`use_browser`/`name_session`/`mark_tab`. Its non-shared `Browser` uses a unique daemon + own browser; its shared `Browser('chrome')` uses the default daemon without an owner key. The upstream "many agents, one Chrome" capability is therefore not in force on the shared path.
- evidence: `browser-use` `pyproject.toml:19`; `browser_use/browser/browser.py:900-940`; source-wide grep.
- missing: a runtime test of two concurrent `Browser('chrome')` agents in one process; a check of whether the host sets `BH_OWNER`.
- next probe: two async `Browser('chrome')` runs in one process, each opening a tab; inspect owner keys and tab ownership.
- why it matters: it explains why the old C-02 risk did not disappear with the upstream fix.

## C-09 · 0.1.15 telemetry trim is an undocumented release delta (Observation)

- status: Observation; diff-confirmed, intent unknown.
- claim: 0.1.15 stops sending the script text, stdout/stderr tails, helper arguments and exception text in `cli_event`; it keeps lengths, helper names, duration and an error boolean. `CHANGELOG.md` has no 0.1.15 section, so users cannot learn this from the release notes.
- evidence: EK-59; `run.py:207-237,323-356`; `telemetry.py:282-315`; 0.1.14 wheel diff in `audit/_scratch/bh0114`.
- missing: the release PR/commit message (not in the shallow tree); whether the trim was privacy-driven or payload-driven.
- next probe: read the 0.1.15 release PR #1077 discussion; compare a real event payload with 0.1.14.
- why it matters: privacy boundary changed without a changelog statement; the package's EK-02 description of "no args" is a 0.1.15 delta, not a 0.1.13 property.
