# Repository Snapshot Artifact — chrome-devtools-axi

- run_id: `ARCH-2026-11-26-001`
- job_id: `KA-2026-11-26-001`
- snapshot_time: 2026-11-26 (Asia/Shanghai)
- snapshot_method: `git clone https://github.com/kunchenguid/chrome-devtools-axi.git` + `pnpm install --frozen-lockfile --ignore-scripts` + `pnpm run build` + `pnpm test`
- local_path: `/home/user/.doubao/agent_mode/workspace/archaeology-jobs/ARCH-2026-11-26-001/repo`
- read_only: yes (only build artifacts and ignored `node_modules`/`dist` were produced locally; `git status --porcelain` was empty after build)

## 1. Provenance

| Field | Value | Evidence |
|---|---|---|
| Repository | `https://github.com/kunchenguid/chrome-devtools-axi.git` | `package.json` `repository.url`; clone URL |
| Branch | `main` | `git rev-parse --abbrev-ref HEAD` |
| HEAD commit | `1dbdac1ca1122e1d2b76d889bbc8a2dd2eb2dcd7` | `git log -1 --format=%H` |
| HEAD date | `2026-11-04T05:32:02+11:00` | `git log -1 --date=iso-strict` |
| HEAD author | Luke Cameron | `git log -1 --format=%an` |
| HEAD subject | `feat(cli): add newpage --context for separate cookie jars (#200)` | `git log -1 --format=%s` |
| `git describe` | `chrome-devtools-axi-v0.1.46-2-g1dbdac1` | `git describe --tags --always` |
| package version | `0.1.46` | `package.json` |
| npm latest | `0.1.46`; `time.modified = 2026-10-29T13:48:43.002Z` | `npm view chrome-devtools-axi version dist-tags.latest time.modified` |
| History depth | 144 commits; `git rev-parse --is-shallow-repository` = `false` | `git rev-list --count HEAD`; clone was requested with `--depth 200`, server returned full history |
| First commits | `360b384` / `ac8389b` `Initial commit`, `2026-03-27` | `git log --reverse` |
| Size | worktree 952K; `.git` 1.3M | `du -sh . .git` |

Version drift note: HEAD is two commits past tag `chrome-devtools-axi-v0.1.46`. The two post-release commits are `newpage --context` work (PR #200 and its predecessor), so the checked-out source is slightly ahead of the published `0.1.46` artifact. All findings below are against the commit SHA, not the npm tarball.

## 2. Size and language map

| Metric | Value | Evidence |
|---|---|---|
| Tracked files | 78 | `git ls-files \| wc -l` |
| TypeScript files | 50 | `git ls-files '*.ts' \| wc -l` |
| TypeScript lines | 17,051 | `git ls-files '*.ts' \| xargs cat \| wc -l` |
| All tracked text lines (excluding directories) | 21,088 | `git ls-files \| xargs wc -l` |
| Test files | 29 `test/*.test.ts` | `ls test/*.test.ts \| wc -l` |
| Static `it(` call sites | 654 | `grep -rFo 'it(' test/*.ts \| wc -l` |
| Executed tests at HEAD | **796 passed / 796** | `pnpm test` at 2026-11-26, 29 files passed, 80.57s |
| Build | **pass** | `pnpm run build` → `tsc && chmod +x dist/bin/chrome-devtools-axi.js` |
| Node used locally | `v22.23.4` | `node --version` (CI pins Node 24) |
| Package manager | `pnpm@11.1.1` | `package.json` `packageManager`; local `pnpm --version` 11.1.1 |

The 796/654 gap is `it.each` table expansion: 654 static `it(` sites expand to 796 executed cases. The earlier working estimate of 637 was a stale static count and is superseded by the executed result above.

Largest files by line count (source and tests):

| File | Lines |
|---|---|
| `src/cli.ts` | 2,023 |
| `src/bridge.ts` | 1,272 |
| `src/client.ts` | 965 |
| `src/run.ts` | 392 |
| `src/pages.ts` | 354 |
| `src/selected-page.ts` | 239 |
| `src/snapshot.ts` | 152 |
| `src/uid-freshness.ts` | 141 |
| `src/sessions.ts` | 122 |
| `src/idle.ts` | 100 |
| `src/navigation-timeout.ts` | 74 |
| `src/skill.ts` | 77 |
| `src/suggestions.ts` | 79 |
| `src/generation.ts` | 56 |
| `src/bridge-script.ts` | 50 |
| `src/version.ts` | 29 |
| `src/paths.ts` | 9 |

## 3. Top-level layout

```
.agents/            agent-facing repo instructions/skills metadata
.airlock/           local review gate: lint.sh + workflows/main.yml
.claude/            Claude-specific skill/agent config
.github/workflows/  ci, release-please, guard-generated-files, no-mistakes-required
bin/                chrome-devtools-axi.ts, chrome-devtools-axi-bridge.ts
scripts/            build-skill.ts
skills/             generated/published skill package
src/                CLI, bridge, client, page/session/snapshot modules
test/               vitest suites + fake MCP fixture
AGENTS.md CLAUDE.md CONTRIBUTING.md README.md VISION.md CHANGELOG.md
package.json pnpm-lock.yaml pnpm-workspace.yaml tsconfig.json
release-please-config.json .release-please-manifest.json
```

## 4. Entrypoints and process topology

- CLI entry: `bin/chrome-devtools-axi.ts` → `src/cli.ts` `main(argv)`.
- Bridge entry: `bin/chrome-devtools-axi-bridge.ts` → `src/bridge.ts` `runBridge()` (starts near `src/bridge.ts:1196`).
- Process topology per `README.md` "How It Works": short-lived CLI → persistent **Bridge Server** on loopback HTTP → `chrome-devtools-mcp` over stdio or Streamable HTTP → Chrome.
- Bridge routes: `GET /health` (shallow), `GET /health?deep=1` (calls `list_pages` to prove CDP target reachable), `GET /tools`, `POST /call`.
- Default bind: `127.0.0.1:9224` for the `default` session.

Evidence: `src/bridge.ts` `handleBridgeRequest` route branches; `src/client.ts` `ensureBridge`; `README.md`.

## 5. Core modules and responsibilities

| Module | Responsibility | Key symbols |
|---|---|---|
| `src/cli.ts` | ~35 command handlers, flag parsing, output formatting, exit codes | `COMMAND_HANDLERS`, `main` |
| `src/client.ts` | CLI→bridge transport, bridge discovery/spawn/health/terminate, `callTool`, `CdpError` | `ensureBridge`, `callTool`, `CdpError`, `stopBridge` |
| `src/bridge.ts` | HTTP server, security gate, MCP transport resolution, MCP client wiring, roots, tool forwarding, keychain args, idle wiring | `runBridge`, `handleBridgeRequest`, `isRequestAllowed`, `resolveTransport`, `buildTransportArgs`, `KEYCHAIN_ISOLATION_CHROME_ARGS` |
| `src/sessions.ts` | session name/port/path derivation | `resolveSessionName`, `resolvePort`, `resolveStateDir` |
| `src/selected-page.ts` | persisted page routing id | `readSelectedPageId`, `writeSelectedPageId`, `clearSelectedPageId` |
| `src/pages.ts` | `list_pages` display parsing; page-scoped tool set | `PAGE_SCOPED_TOOLS`, `needsPageId`, `parsePagesList` |
| `src/snapshot.ts` | snapshot truncation + uid rewrite | `truncateSnapshot`, uid helpers |
| `src/uid-freshness.ts` | generation-aware ref validation, fresh snapshot capture | `parseUidFresh`, `captureFreshSnapshot` |
| `src/generation.ts` | persisted snapshot generation counter + page MutationObserver | `bumpGeneration`, `readGeneration` |
| `src/idle.ts` | opt-in idle shutdown, in-flight deferral, ambient probe exclusion | `createIdleTracker`, `resolveIdleTimeoutMs` |
| `src/navigation-timeout.ts` | derived MCP/HTTP deadlines from navigation timeout | `mcpRequestTimeoutMs`, `callRequestTimeoutMs` |
| `src/run.ts` | stdin script runner, `page` helper, temp `.mjs` import | `runScript`, `createPageHelper`, `wrapJsExpression` |
| `src/paths.ts` | resolve output paths against invoking CLI cwd | `resolveOutputPath` |

## 6. Core data structures and state

- `CdpError` (`src/client.ts`): `{ message, code, suggestions }`; codes `BRIDGE_NOT_READY`, `REF_NOT_FOUND`, `STALE_REF`, `TIMEOUT`, `BROWSER_ERROR`, `VALIDATION_ERROR`, `UNKNOWN`.
- Bridge PID file `bridge.pid`: stores PID and port; used by `ensureBridge` and `stopBridge`.
- `selected-page-id`: last page id selected by `select_page` / `new_page`; consumed by page-id routing.
- `snapshot-generation`: monotonically bumped counter; snapshot uids are rewritten `uid=gN:...`; validation requires same generation and zero observed mutations.
- Session state directory: default `~/.chrome-devtools-axi/`; named sessions `~/.chrome-devtools-axi/sessions/<name>/`.

## 7. Configuration surface

Environment variables observed in source/tests: `CHROME_DEVTOOLS_AXI_SESSION`, `_PORT`, `_MCP_PATH`, `_MCP_SERVER_URL`, `_HEADED`, `_CHROME_ARGS`, `_USER_DATA_DIR`, `_BROWSER_URL`, `_AUTO_CONNECT`, `_CHANNEL`, `_WS_HEADERS`, `_EXECUTABLE_PATH`, `_BRIDGE_TIMEOUT_MS`, `_IDLE_TIMEOUT_MS`, `_NAVIGATION_TIMEOUT_MS`, `_PAGE_ID_ROUTING` (via `pages.ts` docs/args).

Key defaults:
- bridge readiness timeout 30,000 ms (floor 1,000 ms);
- MCP per-request 60,000 ms; CLI `/call` 120,000 ms; headroom 30,000 ms;
- navigation timeout max 1,000,000,000 ms;
- idle off by default, floor 1,000 ms;
- snapshot truncation 16,000 chars unless `--full`.

## 8. Security and governance mechanisms (observed)

- Loopback-only Host/Origin gate before routing: `LOOPBACK_HOSTNAMES = {"127.0.0.1","localhost","::1"}`, `isRequestAllowed`, 403 `{"error":"Forbidden host"}`, logging of host/origin/method/url.
- No authentication token: any local process that can set `Host: 127.0.0.1:<port>` can reach `POST /call`. This is stated in the fixing commit message as an explicit out-of-scope follow-up.
- The gate covers `/health`, `/tools`, `/call`; tests assert no CDP tool is invoked on rejection.
- Keychain isolation flags appended only for launch modes (`--isolated`, `--userDataDir`), never for attach modes.
- Session names validated against traversal/separators/shell metacharacters; port collision exits 48.
- CI: Node 24, frozen lockfile, build, test. `.airlock/lint.sh` runs Prettier + `tsc --noEmit`; review gate risk threshold `medium`. `guard-generated-files.yml` forbids hand edits to `CHANGELOG.md` / manifest. `no-mistakes-required.yml` requires PR body attestation. `release-please.yml` publishes with `--provenance`.

## 9. Tests and what they pin

- `test/bridge.test.ts` (139 static cases): security gate matrix, including missing Host, forged non-loopback Host on all three routes, loopback Host + hostile Origin, unparseable Origin, `[::1]evil.com`, `localhost.evil.com`; asserts CDP not called and rejection logged.
- `test/keychain-isolation.test.ts` (12): launch vs attach keychain-flag split; caller `--chrome-arg` cannot drop isolation.
- `test/bridge-lifecycle.test.ts` (3): real detached bridge against fake MCP; stays up idle by default; exits on idle timeout; exits when MCP server dies.
- `test/idle.test.ts` (11): parsing floor and in-flight deferral/ambient behavior.
- `test/snapshot.test.ts` (28) + `test/generation.test.ts` (1): generation bump and stale-ref behavior.
- `test/selected-page*.test.ts` (27 + 2): page selection persistence and clearing.
- `test/sessions.test.ts` (20): traversal rejection, deterministic ports, state paths.
- `test/navigation-timeout.test.ts` (39): deadline arithmetic.
- `test/isolated-context.test.ts` (11): `newpage --context` argument validation and isolated context forwarding.
- `test/airlock-lint.test.ts` (1): lint script must use pnpm, not npm/npx.
- `test/release-ci-exclusions.test.ts` (6): release-please outputs align with CI path ignores.

## 10. Snapshot conclusion

This is a small, single-package TypeScript CLI/wrapper with a long-running loopback bridge. Its most consequential engineering surface is the bridge's local control plane: unauthenticated HTTP on a fixed loopback port, protected only by Host/Origin allowlisting after GHSA-x439-jhfh-v9x2. All 796 tests and the TypeScript build pass at HEAD in this environment. No browser was launched and no live Chrome/CDP interaction was performed; runtime claims about actual Chrome behavior remain unexecuted in this run.
