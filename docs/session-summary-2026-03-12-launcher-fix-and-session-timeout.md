# Session Summary: Launcher Lockfile Fix, Session Timeout, and Dev Environment Cleanup

**Date:** 2026-03-12 (afternoon)
**Branch:** `dev`
**QMD version:** v2.0.1 (source build)
**Participants:** rymalia + Claude (Opus 4.6)

---

## Purpose

Three goals: (1) implement and test a fix for the `bin/qmd` launcher runtime detection false positive (#381/#380), (2) investigate and fix MCP HTTP session management issues discovered when Gemini CLI reported "stale session ID" errors, and (3) clean up the dev environment so we're definitively running from source — not the published npm package.

---

## Changes Made

| Change | Detail |
|--------|--------|
| **Launcher lockfile priority (#381)** | `bin/qmd` — added `package-lock.json` check before `bun.lock`. If npm's lockfile exists, Node is used regardless of `bun.lock` being in the repo. Fixes source builds getting routed to Bun |
| **Cleanup guard (#380)** | `src/store.ts:cleanupOrphanedVectors()` — replaced `sqlite_master` table existence check with `isSqliteVecAvailable()`. The old check passed even when the vec0 module wasn't loaded (virtual table metadata persists), causing crashes |
| **Launcher detection tests** | `test/launcher-detection.test.sh` — 7 shell test cases covering all lockfile combinations (no lockfiles, bun only, npm only, both, all three) |
| **PR #385 submitted** | `fix/launcher-lockfile-priority` branch → [tobi/qmd#385](https://github.com/tobi/qmd/pull/385). Fixes #381 and #380 |
| **Session idle timeout** | `src/mcp/server.ts` — sessions expire after 30 minutes of inactivity, swept every 60 seconds. Configurable via `sessionIdleTimeout` and `sessionSweepInterval` options |
| **Stale session logging** | 404 "Session not found" responses are now logged with the truncated session ID (were previously silent) |
| **Session lifecycle tests** | `test/mcp.test.ts` — 7 new tests: active session, unknown ID rejection, idle expiry, activity reset, re-initialization after expiry, concurrent session independence, default timeout constant |
| **npm global package removed** | Uninstalled `@tobilu/qmd` from npm global, replaced with `npm link` from source checkout |
| **Stale build artifact removed** | Deleted `dist/mcp.js` — leftover from the `feature/mcp-multi-client` branch build that predated Joel's upstream implementation |

---

## Key Decisions

### Launcher: `package-lock.json` priority (not `.gitignore` or PATH check)

Three options were considered for #381:
1. **Check `package-lock.json` first** (chosen) — positive signal that npm installed deps
2. Add `bun.lock` to `.gitignore` — changes maintainer workflow
3. Check invocation path — too brittle

`package-lock.json` is the right signal because it's generated locally by `npm install` and never committed. Its presence definitively means npm compiled native modules for Node. This is the minimal change that doesn't affect the maintainer's Bun-based workflow.

### Session timeout: 30 minutes (not 5, not unlimited)

- **5 minutes** (from rymalia's earlier feature branch `756c89e`) was too aggressive — killed Gemini CLI sessions during normal LLM thinking pauses
- **No timeout** (Joel's upstream implementation `383a2e5`) caused unbounded session accumulation — 635 sessions after 16 hours
- **30 minutes** is long enough for any realistic LLM reasoning chain, short enough to prevent meaningful memory waste

### Dev environment: source-only, no npm package

Running the published npm package alongside a source checkout caused confusion:
- Two copies of `dist/mcp.js` with different code (one had session timeout, one didn't)
- Unclear which binary the daemon was actually executing
- Stale build artifacts from old branches masquerading as "the installed version"

Resolution: `npm uninstall -g @tobilu/qmd` + `npm link` from source. The global `qmd` command now resolves through a directory symlink: `~/.nvm/.../node_modules/@tobilu/qmd → /Users/rymalia/projects/qmd`. One copy, one source of truth. `npm run build` → immediately live.

---

## Discoveries

### The "installed binary with session timeout" was our own stale artifact

The file `dist/mcp.js` (dated Mar 1) contained session timeout code with a 5-minute TTL. We initially attributed this to the published npm package. It was actually a build artifact from rymalia's `feature/mcp-multi-client` branch (`756c89e`, Feb 25), which was never pushed upstream. The artifact survived because `npm run build` outputs to `dist/mcp/server.js` (preserving `src/` structure), not `dist/mcp.js` (flat bundle). The CLI imports `../mcp/server.js`, so the stale file was dead code but caused diagnostic confusion.

### Joel's upstream implementation omitted session cleanup

Joel Johnson's PR #195 (`383a2e5`, Mar 3) independently implemented the same per-session MCP architecture but without any idle timeout or sweep. This is what's in upstream `main` today. The session timeout was never in the upstream codebase — it only existed on rymalia's unmerged feature branch.

### MCP SDK client does not auto-recover from session expiry

The `@modelcontextprotocol/sdk` client (`StreamableHTTPClientTransport`) handles 401/403 with auth retry and has SSE reconnection logic, but a 404 on a stale session ID just throws `StreamableHTTPError`. What happens next depends on the host application:
- **Claude Code**: apparently catches and re-initializes transparently (users never see session errors)
- **Gemini CLI**: surfaces the error to the agent as "stale session ID"

This is a client-side behavior difference, not something we can fix server-side. The 30-minute timeout makes it rare enough to be a non-issue for interactive use.

### LaunchAgent manages the daemon, not `--daemon` flag

The `--daemon` flag creates a PID file and forks, but the LaunchAgent (`com.qmd.mcp.plist`) with `KeepAlive: true` is what actually manages the daemon lifecycle. Killing the process → LaunchAgent restarts it automatically within seconds. The LaunchAgent logs to `/tmp/qmd-mcp.error.log`; the `--daemon` flag logs to `~/.cache/qmd/mcp.log`.

### Log timestamps are UTC

The `ts()` function uses `new Date().toISOString().slice(11, 23)` which is always UTC. User is in PDT (UTC-7). Not a bug, just needs awareness when reading logs.

---

## Testing

### Test Results

| Suite | Result |
|-------|--------|
| Full vitest suite | **705/705** pass (698 existing + 7 new session lifecycle) |
| Launcher detection (shell) | **7/7** pass |

### Session Lifecycle Tests Added

| Test | What it proves |
|------|---------------|
| `active session responds normally` | Basic session routing works after initialize |
| `unknown session ID returns 404` | Fabricated session IDs are rejected with correct error |
| `idle session expires after timeout` | Sessions are reaped by the sweep after idle timeout |
| `activity resets the idle timer` | Requests within timeout keep session alive (no false expiry during active use) |
| `expired session can re-initialize` | After expiry, new `initialize` creates fresh session |
| `multiple concurrent sessions are independent` | Multi-client isolation works |
| `default timeout is 30 minutes` | Guards against accidentally changing production default |

Tests use a 2-second timeout with 500ms sweep interval for fast execution (~10 seconds for the time-dependent tests).

---

## Current State

- **Running from source**: `/Users/rymalia/projects/qmd` via `npm link`
- **Daemon**: PID managed by LaunchAgent, running with 30-minute session timeout
- **PR #385**: Submitted — launcher lockfile priority + cleanup guard
- **Session timeout**: Implemented and tested but not yet submitted as a PR (needs its own branch)

### Files Modified (uncommitted on `dev`)

| File | Change |
|------|--------|
| `bin/qmd` | `package-lock.json` priority in runtime detection |
| `src/store.ts` | `cleanupOrphanedVectors` uses `isSqliteVecAvailable()` |
| `src/mcp/server.ts` | Session idle timeout (30 min), sweep interval, stale session logging, configurable options |
| `test/mcp.test.ts` | 7 new session lifecycle tests |
| `test/launcher-detection.test.sh` | 7 lockfile combination tests |

### Upstream PRs

| PR/Issue | Title | Status |
|----------|-------|--------|
| [#385](https://github.com/tobi/qmd/pull/385) | fix: prioritize package-lock.json in launcher to prevent Bun false positive | Open, fixes #381 and #380 |

### Potential Follow-Up PRs

- **Session idle timeout** — upstream has no session cleanup at all. Could submit as a feature PR referencing the 635-session accumulation we observed. Needs its own branch off `main`.
- **PR Candidate A** (from `docs/pr-candidates-2026-03-12.md`) — remove `mcpServers` from plugin `marketplace.json`. Still relevant, not started this session.

---

## Recommendations

1. **Always `npm run build` after source changes.** The `npm link` makes the global `qmd` point to your `dist/`, but you still need to compile TypeScript.

2. **Kill the daemon to pick up changes.** After rebuilding, kill the qmd process and the LaunchAgent will restart it with the new code within seconds.

3. **Monitor session lifecycle with:** `tail -f /tmp/qmd-mcp.error.log | grep "expired\|rejected\|active"`

4. **The session timeout PR should be separate from #385.** The launcher fix is a bugfix; the session timeout is a new feature. Different review criteria.

5. **If Gemini CLI complains about stale sessions again**, it's hitting the 30-minute timeout. The fix is on Gemini's side (auto re-initialize on 404), but bumping the timeout higher is an option if 30 minutes proves insufficient.
