# Session Summary: 2026-02-24 - Gemini CLI MCP Integration & Multi-Client Fix

## Strategic Context
This session focused on connecting the QMD MCP HTTP server to **Google Gemini CLI** as a second client alongside Claude Code. This exposed a fundamental architectural limitation: the HTTP server only supported a single MCP client at a time.

## Key Decisions & Rationale

1. **Multi-Client Session Architecture:** Refactored `src/mcp.ts` HTTP transport from a single shared `McpServer` + `WebStandardStreamableHTTPServerTransport` to per-client session management. Each connecting client now gets its own `McpServer` + transport pair, keyed by `Mcp-Session-Id` header.
   - **Rationale:** The MCP protocol's `initialize` handshake is stateful — a server can only be initialized once. Multiple clients need independent protocol state while sharing the underlying search index.

2. **Session Idle Timeout:** Set to 5 minutes with 60-second cleanup sweeps.
   - **Observation:** This may be too aggressive — both Claude Code and Gemini CLI sessions expired and had to re-initialize during testing. Consider bumping to 15-30 minutes.

3. **Bypassed Build for Testing:** Used `npx tsx src/qmd.ts mcp --http` to run the MCP server from TypeScript source directly, avoiding the broken `tsc` build.
   - **Rationale:** Pre-existing Zod v3/v4 type mismatch with `@modelcontextprotocol/sdk` prevents `tsc` compilation. Runtime behavior is correct.

## Changes Made

| Change | Detail |
|--------|--------|
| **Multi-client MCP sessions** | `src/mcp.ts` — replaced single transport with per-session `Map<string, Session>` keyed by UUID. Each `POST /mcp` without a session ID creates a new `McpServer` + transport. Stale session IDs return 400. |
| **Session cleanup timer** | `src/mcp.ts` — 60-second interval expires sessions idle >5 min. Cleanup timer cleared on server stop. |
| **Session-aware non-POST handler** | `src/mcp.ts` — GET/DELETE `/mcp` routes now require valid session ID. DELETE cleans up session. |
| **Graceful shutdown** | `src/mcp.ts` — `stop()` now clears interval, closes all session transports, and clears the session map. |

## Gemini CLI Configuration

Working config at `~/.gemini/settings.json`:
```json
{
  "mcpServers": {
    "qmd": {
      "httpUrl": "http://localhost:8181/mcp",
      "trust": true
    }
  }
}
```

Gemini CLI commands for MCP management:
- `/mcp list` — list configured servers
- `/mcp desc` — list with tool descriptions
- `/mcp refresh` — restart all MCP connections
- `/mcp enable <name>` / `/mcp disable <name>` — toggle servers

## Issues Discovered

### 1. Build Broken — Zod Type Mismatch (Pre-existing)
**Symptom:** `bun run build` fails with ~12 errors like `ZodString is not assignable to type AnySchema`.
**Cause:** `@modelcontextprotocol/sdk` expects Zod v3 types but the project has Zod v4 (or vice versa). The `inputSchema` fields in tool registrations fail type assignment.
**Impact:** Cannot compile TypeScript to `dist/`, which means:
- Cannot `npm link` to install local version globally
- Cannot publish new npm release
- The globally installed `qmd` CLI stays at v1.0.7
**Workaround:** Run from source with `npx tsx src/qmd.ts <command>`.

### 2. Installed qmd v1.0.7 Behind Local v1.1.0
**Symptom:** `qmd query "threaded tw expansion"` crashed with reranker context size exceeded (needs 2932, limit 2048).
**Cause:** Commit `5233e67` (`fix(rerank): truncate documents exceeding 2048-token context size`) is in local source but not in the installed npm package.
**Impact:** CLI users on v1.0.7 hit crashes on long documents. The crash also triggers a Metal GPU cleanup race condition (`GGML_ASSERT` failure in llama.cpp).
**Fix path:** Resolve the Zod build issue, then publish v1.1.0.

### 3. Memory Pressure on M3 with Concurrent Searches
**Symptom:** Music player stutters, mouse freezes during concurrent vector searches.
**Cause:** QMD loads ~2GB of models into unified memory. With HTTP multi-client, two searches can run simultaneously, saturating GPU compute on the M3's shared memory architecture.
**Potential mitigations:**
- Request queuing/serialization in the HTTP server (only one heavy search at a time)
- Lower `RERANK_CANDIDATE_LIMIT` from 40 to reduce GPU workload
- Add a concurrency semaphore around LLM operations

## Environment State

- **MCP HTTP server:** Running from source via `npx tsx src/qmd.ts mcp --http` (background process)
- **LaunchAgent:** Stopped via `launchctl bootout`. Will need to be re-bootstrapped when switching back to the daemon.
- **Installed CLI:** v1.0.7 at `~/.nvm/versions/node/v24.12.0/bin/qmd` — missing recent fixes
- **Local source:** v1.1.0 — has multi-client fix + rerank truncation fix

## How to Run (Current State)

```bash
# MCP HTTP server (from source, foreground)
npx tsx src/qmd.ts mcp --http

# CLI queries (from source, bypasses stale v1.0.7)
npx tsx src/qmd.ts query "your query"

# Restart the LaunchAgent daemon (uses old dist/, NOT recommended until build fixed)
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.qmd.mcp.plist
```

## Unfinished Work & Next Steps

### 1. Fix Zod Build Compatibility (Blocking)
The `tsc` build fails due to Zod v3/v4 type mismatch with `@modelcontextprotocol/sdk`. Options:
- Pin Zod to the version the MCP SDK expects
- Update `@modelcontextprotocol/sdk` to a version compatible with current Zod
- Add type assertions (`as any`) to the `inputSchema` fields as a quick workaround
- **This blocks publishing v1.1.0 and `npm link`**

### 2. Publish v1.1.0 Release
Once the build works, cut a release to get the rerank truncation fix and multi-client HTTP support into the installed CLI. Includes:
- Multi-client MCP HTTP sessions (this session)
- Rerank context size truncation fix (`5233e67`)
- Any other unreleased commits since v1.0.7

### 3. Tune Session Idle Timeout
Current 5-minute timeout causes frequent re-initialization. Consider:
- Bumping to 15-30 minutes
- Or making it configurable via CLI flag (`--session-timeout`)

### 4. Consider Concurrency Controls for M3
Add a semaphore or request queue to prevent concurrent heavy LLM operations (embedding, reranking, query expansion) from saturating GPU memory on constrained hardware.

### 5. Update LaunchAgent to Run from Source
The LaunchAgent plist (`~/Library/LaunchAgents/com.qmd.mcp.plist`) points to the installed `qmd` binary. Once the build/release is done, either:
- Update it to point to the new version
- Or temporarily change it to use `npx tsx` from the project directory

## Summary Statistics
- **Files Modified:** 1 (`src/mcp.ts`)
- **Net Change:** ~60 lines added (session map, cleanup, routing logic)
- **New Type Errors:** 0 (27 pre-existing, 27 after)
- **Clients Tested:** Claude Code MCP + Gemini CLI MCP (both connected successfully)
