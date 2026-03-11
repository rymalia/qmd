# Session Summary: LaunchAgent v1.1.0 Deployment & Multi-Client Fix

**Date**: 2026-03-01
**Agents**: Claude Code (primary), Gemini CLI (troubleshooter)

## TL;DR

The QMD MCP server had been failing for Gemini CLI with `"Server already initialized"` errors. Root cause: the macOS LaunchAgent was still running the v1.0.7 global binary, which lacks multi-client session support. The multi-client fix existed on an unmerged feature branch (`feature/mcp-multi-client`). Merged it, rebuilt, relinked, and restarted the LaunchAgent with the correct binary path. Also killed 14 orphaned stdio qmd processes and updated documentation.

## Root Cause Analysis

### The Symptom
Gemini CLI could not connect to the QMD MCP HTTP server on port 8181:
```
{"code":-32600,"message":"Invalid Request: Server already initialized"}
```

### What Actually Happened (Three-Layer Problem)

**Layer 1: Wrong binary in LaunchAgent**
The LaunchAgent `com.qmd.mcp.plist` was pointing to `~/.nvm/versions/node/v24.12.0/bin/qmd` — the globally installed v1.0.7 from npm. This binary lacks the multi-client session management added in the Feb 24 session. Every time the LaunchAgent restarted (including after `KeepAlive` triggers), it spawned the old single-client v1.0.7.

**Layer 2: Unmerged feature branch**
The multi-client fix (commit `756c89e`) was on `feature/mcp-multi-client` but was never merged into `dev`. The branch topology:
```
* 7bc12e2 (dev) ← docs-only commits
* 652fecc
* 3430733
| * 756c89e (feature/mcp-multi-client) ← actual code fix
|/
* ef847a4
```
The Feb 24 session generated four session summary documents describing the fix as "implemented" and "verified," but the actual code change sat on an unmerged branch.

**Layer 3: Gemini agent's partial fix**
The Gemini agent correctly diagnosed the v1.0.7 binary problem and updated the LaunchAgent to use `node_modules/.bin/tsx` pointing to local source. This appeared to work because:
- Restarting the server cleared the stale single-session state
- Gemini CLI connected as the first (and only) client
- Multi-client was never actually retested

But the `dev` branch source also lacked the multi-client code, so the tsx approach would have failed on the next simultaneous connection. The Gemini agent also introduced a fragile dependency: `node_modules/.bin/tsx` is a symlink that gets wiped by `bun install`.

### Why It Wasn't Caught Earlier

1. **Session summaries described work as complete** — four docs from Feb 24-25 described the multi-client fix as implemented, tested, and working. No doc mentioned it was on an unmerged branch.
2. **The build blocker masked the deployment gap** — the Zod type mismatch prevented `tsc` compilation until commit `652fecc` pinned Zod to 4.2.1. By the time the build worked, nobody circled back to check whether the multi-client code was in `dev`.
3. **`bun run build` + `bun link` was never run** — even after the Zod fix unblocked the build, the global binary was never updated from v1.0.7 to v1.1.0.
4. **LaunchAgent auto-restart hid the problem** — `KeepAlive: true` means the LaunchAgent silently restarts the old binary after any kill. The service always looked "healthy" via `/health` because the v1.0.7 binary starts fine — it just can't handle a second client.

## What We Fixed

| Change | Detail |
|--------|--------|
| **Merged feature branch** | `git merge feature/mcp-multi-client` into `dev` — brings per-session `Map<string, Session>` architecture into the main branch |
| **Built and linked v1.1.0** | `bun run build && bun link` — global `~/.bun/bin/qmd` now points to compiled v1.1.0 with multi-client support |
| **Updated LaunchAgent binary** | `com.qmd.mcp.plist` now uses `~/.bun/bin/qmd` (stable symlink via `bun link`) instead of `~/.nvm/.../bin/qmd` (stale npm global) or `node_modules/.bin/tsx` (fragile) |
| **Killed 14 orphaned processes** | Old stdio `node ~/.bun/bin/qmd mcp` processes from past Claude Code sessions (Wed-Sat) were consuming memory |
| **Updated CLAUDE.md** | Changed LaunchAgent binary path from nvm to bun; added "Deploying source changes" section with build-link-restart workflow |
| **Updated GEMINI.md** | Added deploy workflow note and multi-client documentation |

## Verification

```
# Two simultaneous initializes — both succeed
Client 1 (claude-code): SUCCESS
Client 2 (gemini-cli):  SUCCESS

# Health check
{"status":"ok","uptime":5}

# Process using correct binary
node /Users/rymalia/.bun/bin/qmd mcp --http

# Version
qmd 1.1.0 (fa3cf12)
```

## The Deployment Pipeline (Now Documented)

The full path from source change to running LaunchAgent:

```
src/*.ts  →  bun run build  →  dist/*.js  →  bun link  →  ~/.bun/bin/qmd  →  LaunchAgent
```

**Each step is necessary.** Missing any one leaves the LaunchAgent running stale code:
- Skip `bun run build` → `dist/` has old JavaScript
- Skip `bun link` → global symlink points to old `dist/`
- Skip LaunchAgent restart → running process uses old code from memory

## Files Modified

| File | Action |
|------|--------|
| `src/mcp.ts` | Merged from `feature/mcp-multi-client` — per-session Map architecture |
| `dist/*.js` | Rebuilt from source |
| `CLAUDE.md` | Updated LaunchAgent binary path; added deploy workflow section |
| `GEMINI.md` | Added deploy workflow note and multi-client documentation |
| `~/Library/LaunchAgents/com.qmd.mcp.plist` | Updated binary from nvm path to `~/.bun/bin/qmd` |

## Lessons Learned

### 1. Feature branches need to be merged, not just documented
Four session summaries described the multi-client fix as "done" but the code sat on `feature/mcp-multi-client`. **A feature isn't shipped until it's on the main branch, built, linked, and running.** Session summaries should include a "Merge Status" field.

### 2. The build-link-restart pipeline has no automation
There's no hook, script, or CI that connects "source changed" to "LaunchAgent is running new code." Every step is manual. This is fine for now but means any code change requires remembering three separate commands.

### 3. LaunchAgent `KeepAlive` masks version problems
Because the LaunchAgent auto-restarts, the service always looks healthy. You can't tell from `/health` whether you're running v1.0.7 or v1.1.0. The version endpoint in the MCP initialize response (`serverInfo.version`) is the only way to verify.

### 4. `node_modules/.bin/` is not a stable binary path
The Gemini agent's fix of pointing the LaunchAgent to `node_modules/.bin/tsx` would have broken on the next `bun install`. LaunchAgents need stable paths: global installs (`~/.bun/bin/`), system binaries (`/usr/local/bin/`), or absolute paths to versioned installations.

### 5. Kill commands must be precise
`pkill -f 'node.*\.bun/bin/qmd mcp'` correctly killed the 14 orphaned stdio processes without touching the LaunchAgent's tsx process (which had a different command pattern). When multiple process types share similar names, the kill pattern matters.

## Current State

- **MCP HTTP server**: Running v1.1.0 (`fa3cf12`) on port 8181 via LaunchAgent
- **Multi-client**: Verified — Claude Code and Gemini CLI can connect simultaneously
- **Global binary**: `~/.bun/bin/qmd` → v1.1.0 via `bun link`
- **LaunchAgent**: Using `~/.bun/bin/qmd mcp --http`, `KeepAlive: true`
- **Orphaned processes**: All killed (14 stdio processes from Wed-Sat)
- **Git**: On `dev` branch, merge commit `fa3cf12`

## Unfinished Work

1. **Commit the doc updates** — CLAUDE.md, GEMINI.md, and README.md changes are unstaged
2. **Session idle timeout** — Still at 5 minutes, previous sessions recommended bumping to 15-30 min
3. **Consider adding version to `/health` response** — Would make it trivial to verify which version the LaunchAgent is running without doing a full MCP initialize
4. **`feature/mcp-multi-client` branch cleanup** — Can be deleted now that it's merged
