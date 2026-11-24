# Reconciliation — Audit Findings to Package Corrections

- **Run**: `ARCH-2026-11-25-001`
- **Audit input**: `06_validation/independent_audit.md` (24 findings)
- **Rule**: corrections modify Package artifacts only; target repository, KnowlegeMap, and production Skill remain untouched.

## 1. Disposition summary

| Audit judgment | Count | Disposition |
|---|---:|---|
| CONFIRMED | 16 | No change |
| PARTIALLY_CONFIRMED | 4 | Scope tightened in Package |
| OVER_GENERALIZED | 1 | Claim narrowed; exception added |
| DOWNGRADED | 1 | Radar card data remains downgraded; repo facts win |
| MISSING | 0 | None |
| CONTRADICTED | 0 | None |
| NEEDS_HUMAN_REVIEW | 2 | Kept as Candidate, not upgraded |

## 2. Corrections applied

### 2.1 EK-16 — physical text mismatch exception (AUD-14, O-01)

**Before**: "A mismatch is `ACTION_FAILED`; an unavailable verification is `delivered_unverified`..."

**After**: "A mismatch is `ACTION_FAILED` **except** physical text mismatch (`typed_by_keys`), which is downgraded to `delivered_unverified` with `verification_scope:unavailable` and `verification_reason:postcondition_mismatch`. Missing evidence (`verified.is_none()`) is also `delivered_unverified` with `insufficient_evidence` or `secure_field`."

**Evidence**: `crates/core/src/actions/post_action.rs` lines 103–138: `physical_text_mismatch` branch returns `Ok(result)` after `with_unverified_delivery`; only the later `verified == Some(false)` branch returns `verification_error(..., mismatch_error(...))`.

### 2.2 KO-07 — value exclusion scoped (AUD-07, O-02)

**Before**: "identity compares only role/native-id/stable-name/description; value and bounds are excluded."

**After**: value and bounds are never predicate fields; label-like roles may promote a value into stable-name via `stable_name(role, name, value)`. The invariant is narrowed from "never depend on value" to "no mutable geometry or freely varying value in the predicate; the only value use is a role-programmed label promotion."

**Evidence**: `crates/core/src/refs/ref_identity.rs` (`stable_name`), `refs/ref_full_predicate.rs` (predicate fields), `refs/ref_capture_proof.rs` (census).

### 2.3 KO-08 — lease scope qualified (AUD-08, O-03)

**Before**: "the interaction lease is machine-wide per user."

**After**: Unix per-user machine-wide (`/tmp/agent-desktop-<uid>/interaction.lock`); Windows uses a named handle; macOS uses the Unix path. There is no single cross-platform lease object.

**Evidence**: `crates/core/src/session/interaction_lease.rs`; `crates/macos/src/system/adapter.rs`; `crates/windows/src/system/interaction_lease/`; `crates/linux/src/system/adapter.rs`.

### 2.4 EK-07 — cluster attachment (free-floating EK)

**Before**: EK-07 (FFI as a second adapter consumer) was not in any KO cluster; `00_overview.md` claimed 0 free-floating.

**After**: EK-07 is added to KO-01's R3 invariant cluster. KO-01 now has six members: EK-01, EK-02, EK-07, EK-29, EK-34, EK-39. All 39 EK are covered; average cluster size is 52/13 = 4.0.

**Evidence**: `crates/ffi/src/lib.rs` (C ABI over `PlatformAdapter`); `crates/core/src/adapter/mod.rs`; `src/main.rs:build_adapter`.

### 2.5 Evidence ledger — EV reuse and counting

- Added **EV-046** (`README.md` FFI/ABI docs); reused existing **EV-045** (`src/cli/mod.rs:is_mutating`).
- Replaced misused EV-037/EK-10 and EV-037/EK-37 references with EV-014 (`Action::base_interaction_policy`).
- Replaced misused EV-038 references for `is_mutating` with EV-045.
- EV-037 now means only the three release guards; EV-038 means only the envelope/drift/doc bundle.
- Evidence rows: 45 → 46. Class table corrected to actual counts: 40 code, 1 code/test, 1 test/doc, 1 governance, 1 doc/code, 2 doc, 1 code/doc/test.

### 2.6 Overview metrics — counted, not estimated

| Metric | Old (claimed) | Corrected | Basis |
|---|---:|---:|---|
| Typed EK links | 93 | **118** | typed `links` entries |
| Average KO cluster size | 3.0 | **4.0** | 52/13 |
| Evidence anchors | 45 | **46** | EV rows |
| Source files read directly | 63 | **removed** | not reproducible from ledger |
| Free-floating EK | 0/39 | **0/39** | after EK-07 attachment |

## 3. Findings kept as Candidate (not upgraded)

- **CAND-10** (`read_label` eagerly evaluates `read_name` even for secure targets): code-level behavior confirmed; severity and provider-side consequences are not measured. Stays `NEEDS_HUMAN_REVIEW`.
- **CAND-11** (Linux X11/Wayland matrix): code and docs agree; no display server was available to execute. Stays `PARTIALLY_CONFIRMED`, not runtime-verified.
- **CAND-01** (clipboard outside the permission map): code fact confirmed; whether this is a security gap is a threat-model decision, not an archaeology fact.

## 4. Findings rejected or contradicted

None. The audit found no direct code contradiction. The radar card's `1.2.0 / 60 / 56` remains downgraded in favor of repository `1.3.1 / 61 / 57`; the `~50k weekly downloads` remains unverified and is not used as a fact.

## 5. Gate decision

- **Validation gate**: `PARTIAL`.
- **Reason**: no runtime execution, no GUI session, no `cargo test`, no E2E, no MCP child process. The Package is strong on static truth and flow traceability, weaker on runtime coverage.
- **Corpus write**: approved, because no Contradictions were found and all material overgeneralizations were corrected. The unresolved runtime limits are disclosed in `00_overview.md`, `run_metadata.yaml`, and Candidates.

## 6. Immutability check

- Target repo `agent-desktop`: not modified; clone is read-only in this run.
- `KnowlegeMap`: not modified.
- Production `knowledge-archaeology-skill`: not modified; no Skill Evolution proposal.
- Prior Corpus projects: not modified; this run adds only `projects/agent-desktop/`.
