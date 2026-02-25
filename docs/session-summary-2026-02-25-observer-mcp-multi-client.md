# Session Summary: Observer Agent — MCP Multi-Client Fix & Gemini CLI Integration

**Date**: 2026-02-25
**Role**: Observer/reviewer agent in a parallel session, monitoring and independently verifying the primary agent's work on the Gemini CLI MCP integration.

## TL;DR

Monitored a parallel session fixing QMD's MCP HTTP server to support multiple concurrent clients (Claude Code + Gemini CLI). Independently verified the fix works via curl tests. Discovered and confirmed a Bun limitation with sqlite-vec that explains why the installed CLI uses Node.js. Identified several follow-up items and open questions for the primary agent.

## What I Monitored

The user relayed transcript excerpts from a primary agent session that was:
1. Diagnosing the "Server already initialized" error when Gemini CLI tried to connect
2. Implementing per-client session management in `src/mcp.ts`
3. Debugging a memory overload crash from concurrent GPU workloads
4. Discovering the installed v1.0.7 lacks the rerank truncation fix from v1.1.0

## What I Tested & Verified

| Test | Result | Notes |
|------|--------|-------|
| **Health endpoint** | `{"status":"ok","uptime":576}` | Server running after fix deployed |
| **First MCP initialize (curl)** | 200 OK, session `e7fa7c7d…` | New session created correctly |
| **Second MCP initialize (curl, no session header)** | 200 OK, session `92e808c8…` | Confirmed multi-client: two independent sessions, no collision |
| **Missing Accept header** | 406 Not Acceptable | SDK correctly enforces `Accept: application/json, text/event-stream` |
| **`bun src/qmd.ts status`** | Works | Bun handles non-vector SQLite operations fine |
| **`bun src/qmd.ts search`** (BM25) | Works | FTS5 is built into SQLite, no extension needed |
| **`bun src/qmd.ts vsearch`** (vector) | **Crashes** — `SQLiteError: no such module: vec0` then segfault | Confirmed: Bun's `bun:sqlite` cannot load the sqlite-vec dynamic extension |

## Key Findings

### 1. The Multi-Client Fix Is Correct

The architectural approach — per-client `McpServer` + `WebStandardStreamableHTTPServerTransport` pairs sharing a single `createStore()` — is sound. My independent curl tests confirmed two simultaneous sessions with different `mcp-session-id` values, both getting 200 OK responses to `initialize`. The old code would have returned `-32600` on the second call.

### 2. Bun Cannot Do Vector Search (sqlite-vec Extension Loading)

This is a genuine Bun limitation, not a QMD bug. Bun's built-in `bun:sqlite` module does not support `.loadExtension()`, which is required for the `vec0` virtual table from sqlite-vec. This explains the project's seemingly contradictory setup:

- **Development (Bun)**: `bun src/qmd.ts` for non-vector commands, fast iteration
- **Production (Node.js)**: The installed `qmd` binary runs `dist/qmd.js` under Node.js via `better-sqlite3`, which supports dynamic extensions

The CLAUDE.md says "Use Bun instead of Node.js" but this is for the package manager and general development, not a universal runtime claim. Vector search requires Node.js.

### 3. Three Runtimes, Three Capabilities

| Runtime | FTS/BM25 | Vector Search | Rerank | Source |
|---------|----------|---------------|--------|--------|
| `bun src/qmd.ts` | Works | **No** (vec0 missing) | Works | Local v1.1.0 |
| `npx tsx src/qmd.ts` | Works | Works | Works | Local v1.1.0 |
| `qmd` (installed) | Works | Works | **Crashes on long docs** | npm v1.0.7 |

### 4. The LaunchAgent Is Currently Running Old Code

The LaunchAgent at `~/Library/LaunchAgents/com.qmd.mcp.plist` points to `~/.nvm/versions/node/v24.12.0/bin/qmd` (the installed v1.0.7). The primary agent bypassed this by running `npx tsx src/qmd.ts mcp --http` in the foreground. The LaunchAgent needs to be updated once the build/release issue is resolved.

## Concerns I Had & How They Resolved

| Concern | Resolution |
|---------|------------|
| Does Gemini CLI send `mcp-session-id` correctly? | Yes — both clients connected successfully per the primary agent's report |
| Could the shared SQLite store cause write contention? | WAL mode handles concurrent reads. Writes (llm_cache) are infrequent and SQLite serializes them internally. Not a practical concern. |
| Is the 5-minute session timeout too aggressive? | **Yes** — the primary agent observed sessions expiring and needing re-initialization. They recommend 15-30 min. |
| Would Gemini CLI's Streamable HTTP implementation have quirks? | Found GitHub issue #5268 (connection failures), resolved in v0.1.16+. User's version appears recent enough. |

## Questions for the Primary Agent

1. **McpServer cleanup on session expiry**: The session cleanup interval calls `s.transport.close()` but not `s.server.close()`. Does `McpServer` have a `close()` method, and should it be called to free internal listeners/handlers? Leaking McpServer instances could accumulate over time.

2. **Zod build issue specifics**: You mentioned ~12 Zod type errors blocking `tsc`. Is this from a recent `@modelcontextprotocol/sdk` upgrade, or has the build been broken for a while? What version of the SDK is in `package.json`? The quick fix might be pinning the SDK to a compatible version rather than fighting Zod.

3. **Did you test session reconnection after expiry?** When a client's session expires (5-min timeout) and it sends a request with the stale session ID, the server returns 400 "Session expired." Does Gemini CLI handle this gracefully by re-initializing, or does it give up? Claude Code?

4. **`npx tsx` vs the LaunchAgent**: The current foreground `npx tsx` process will die if the terminal closes. Is there a plan to either (a) update the LaunchAgent plist to use `npx tsx`, or (b) fix the build so the LaunchAgent can use the compiled `dist/`?

5. **Concurrency semaphore for LLM ops**: You mentioned M3 memory pressure from concurrent searches. Did you consider adding a simple semaphore (e.g., `let active = 0; if (active >= 1) await queue`) around the heavy LLM operations in `store.ts` or `llm.ts`? This would serialize expensive GPU work without changing the multi-client session architecture.

6. **Was the `sessionIdGenerator` behavior verified?** The transport's `sessionIdGenerator` is called when it processes the `initialize` request. Since we pre-generate the UUID and pass `() => sessionId`, the transport's internal session tracking should match our external `sessions` Map. But did you verify the transport actually uses this ID (not generating its own)?

7. **The build script adds a shebang**: `npm run build` does `tsc` + prepends `#!/usr/bin/env node`. If you `npm link` after building, the global `qmd` will run under whatever `node` is first in PATH (nvm's v24). This is fine now, but if the user upgrades nvm, the native modules break again (the Node 101 ABI problem). Is there any plan to use `#!/opt/homebrew/bin/node` explicitly?

## Noteworthy Observations

### The "Parallel Dimensions" Problem Strikes Again

The user's initial confusion about `npm run build` vs `bun run build` is the same class of problem documented in `docs/node-101-package-managers.md`. In this case, both commands do the same thing (read `package.json`'s `build` script and execute it), but the convention matters for muscle memory and documentation consistency.

### Bun's sqlite-vec Limitation Should Be Documented

The CLAUDE.md says "Use Bun instead of Node.js" without qualifying that vector search requires Node.js. This will confuse future contributors. The CLAUDE.local.md's development section says `bun src/qmd.ts <command>` without noting the vec0 limitation. Suggested additions:

- CLAUDE.md: Add a note that `bun src/qmd.ts` works for non-vector commands; use `npx tsx src/qmd.ts` for full pipeline including vector search
- Node 101 doc: Add a section on "Bun SQLite limitations — dynamic extensions"

### The Session Summary From the Other Agent Is Comprehensive

`docs/session-summary-2026-02-24-gemini-mcp-multi-client.md` covers the implementation details well. My document focuses on the verification/review angle and open questions that weren't addressed.

## Unfinished Work & Follow-Up Tasks

### Blocking
1. **Fix Zod/tsc build** — Cannot publish v1.1.0 or `npm link` until this is resolved
2. **Publish v1.1.0** — Gets rerank truncation fix + multi-client sessions into the installed CLI
3. **Update LaunchAgent** — Currently stopped; needs to point to working binary after build is fixed

### Important but Not Blocking
4. **Bump session idle timeout** — 5 min → 15-30 min to reduce unnecessary re-initialization
5. **Add LLM concurrency semaphore** — Prevent concurrent GPU-heavy operations from freezing the machine on M3
6. **Document Bun's vec0 limitation** — Update CLAUDE.md and Node 101 doc
7. **Verify McpServer cleanup** — Check if `server.close()` is needed alongside `transport.close()` in session expiry

### Nice to Have
8. **Session metrics endpoint** — Add active session count to `/health` response for debugging multi-client issues
9. **Configurable session timeout** — `qmd mcp --http --session-timeout 1800` (seconds)
10. **Consider Bun's upcoming SQLite extension support** — Track if future Bun versions add `.loadExtension()` to `bun:sqlite`, which would eliminate the need for `npx tsx`

## Open Questions

1. Has anyone tested what happens when the MCP daemon crashes mid-session? Do clients reconnect automatically?
2. Is the `enableJsonResponse: true` setting on the transport compatible with all MCP clients, or do some expect SSE?
3. The Gemini CLI config has `"trust": true` — what does this flag control? Is it required for local servers?
4. Should the `/health` endpoint report session count and per-session last-access times for debugging?
5. If two clients trigger deep_search simultaneously, what's the actual VRAM peak? Is 8GB M3 sufficient or does it need 16GB+?

---

*This document was written by an observer agent in a parallel Claude Code session. The primary implementation work was done by a separate agent whose session summary is at `docs/session-summary-2026-02-24-gemini-mcp-multi-client.md`.*
