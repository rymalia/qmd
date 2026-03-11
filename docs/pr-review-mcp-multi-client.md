# PR Review: `feature/mcp-multi-client` → `main`

**Date**: 2026-03-01
**Branch**: `feature/mcp-multi-client` (3 commits ahead of `main`)
**Reviewers**: Code Reviewer, Silent Failure Hunter, Comment/Docs Analyzer

## Scope

| File | Type | Changes |
|------|------|---------|
| `src/mcp.ts` | Code | +78/-9 — per-session Map architecture for multi-client MCP |
| `CLAUDE.md` | Docs | +73 — LaunchAgent binary path, deploy workflow, multi-client note |
| `README.md` | Docs | +130 — REST API section, session management, troubleshooting |
| `GEMINI.md` | Docs | New file — project instructions for Gemini CLI agent |
| `bun.lock` | Config | Dependency lock update |
| `docs/node-101-package-managers.md` | Docs | +296 — Node.js ecosystem education |
| `docs/session-summary-2026-02-24-mcp-http-migration.md` | Docs | +189 — Private session summary |
| `docs/session-summary-2026-03-01-launchagent-v1.1.0-deploy.md` | Docs | +132 — Private session summary |

---

## Critical Issues (3 — must fix before merge)

### 1. Unresolved merge conflict markers in CLAUDE.md and README.md

**Severity**: Critical (broken files)

The cherry-pick of commit `29209f7` onto the feature branch left unresolved `<<<<<<<` / `=======` / `>>>>>>>` conflict markers in both files. These render as raw diff syntax in any markdown viewer.

- **CLAUDE.md lines 168–203**: Both the old nvm binary path and the new bun binary path are present simultaneously inside conflict markers.
- **README.md lines 140–269**: The entire REST API and Session Management sections are inside a conflict block.

**Fix**: Check out the feature branch, resolve by accepting the `29209f7` (incoming) side, and amend or create a new commit.

```bash
git checkout feature/mcp-multi-client
# Manually resolve conflicts in CLAUDE.md and README.md
# Or: git checkout dev -- CLAUDE.md README.md
git add CLAUDE.md README.md
git commit -m "fix: resolve merge conflict markers in docs"
git push
```

---

### 2. Session creation has no try/catch — orphaned transport on failure

**Severity**: Critical (resource leak + protocol violation)
**Location**: `src/mcp.ts` lines 699–711

```typescript
if (!session) {
  const sessionId = randomUUID();
  const sessionTransport = new WebStandardStreamableHTTPServerTransport({
    sessionIdGenerator: () => sessionId,
    enableJsonResponse: true,
  });
  const sessionServer = createMcpServer(store);
  await sessionServer.connect(sessionTransport);  // ← if this throws...
  session = { server: sessionServer, transport: sessionTransport, lastAccess: Date.now() };
  sessions.set(sessionId, session);
}
```

**Problem**: If `createMcpServer()` or `connect()` throws:
1. The partially-constructed `sessionTransport` is never closed (resource leak)
2. The outer catch block returns plain-text `500 Internal Server Error` (not a valid JSON-RPC response)
3. No rate-limiting or circuit breaking on repeated creation failures

**Recommended fix**:
```typescript
if (!session) {
  const sessionId = randomUUID();
  const sessionTransport = new WebStandardStreamableHTTPServerTransport({
    sessionIdGenerator: () => sessionId,
    enableJsonResponse: true,
  });
  try {
    const sessionServer = createMcpServer(store);
    await sessionServer.connect(sessionTransport);
    session = { server: sessionServer, transport: sessionTransport, lastAccess: Date.now() };
    sessions.set(sessionId, session);
    log(`${ts()} New MCP session ${sessionId.slice(0, 8)}…`);
  } catch (err) {
    console.error(`Failed to create MCP session ${sessionId.slice(0, 8)}:`, err);
    await sessionTransport.close().catch((closeErr) => {
      console.error(`Also failed to close orphaned transport:`, closeErr);
    });
    nodeRes.writeHead(503, { "Content-Type": "application/json" });
    nodeRes.end(JSON.stringify({
      jsonrpc: "2.0",
      error: { code: -32603, message: "Server failed to initialize session. Try reconnecting." },
      id: body.id ?? null,
    }));
    return;
  }
}
```

---

### 3. Cleanup interval can close a transport mid-request

**Severity**: Critical (data corruption under load)
**Location**: `src/mcp.ts` lines 602–612 (cleanup) vs. line 713 (`lastAccess` update)

**Problem**: The cleanup interval fires every 60 seconds and checks `lastAccess`, which is set at request *start* (line 713). A slow request (e.g., 3-4s reranking) suspends at `await handleRequest()`, yielding to the event loop. If the cleanup interval fires during that window, it sees the session as idle (lastAccess is stale) and calls `transport.close()`, destroying the transport while `handleRequest` is still using it.

**Recommended fix**: Track in-flight requests per session and update `lastAccess` at request *end*:

```typescript
type Session = {
  server: McpServer;
  transport: WebStandardStreamableHTTPServerTransport;
  lastAccess: number;
  activeRequests: number;  // added
};

// In cleanup — skip sessions with active requests:
if (now - s.lastAccess > SESSION_IDLE_TIMEOUT && s.activeRequests === 0) {
  // safe to close
}

// In request handlers:
session.activeRequests++;
try {
  const response = await session.transport.handleRequest(request, { parsedBody: body });
  // ... write response ...
} finally {
  session.activeRequests--;
  session.lastAccess = Date.now();  // update at END, not start
}
```

---

## Important Issues (5 — should fix)

### 4. All `.catch(() => {})` patterns suppress errors silently

**Severity**: Important (invisible resource leaks)
**Locations**: Lines 607, 749, 780–782

Three separate instances of `transport.close().catch(() => {})` swallow errors with zero logging:

| Location | Context | Risk |
|----------|---------|------|
| Line 607 | Idle session cleanup | Leaked transport holds file descriptors/sockets |
| Line 749 | Client-initiated DELETE | Operator has no visibility into close failures |
| Lines 780–782 | Graceful shutdown | Post-mortem analysis impossible |

**Fix**: Replace all three with `.catch((err) => { console.error(...) })`.

---

### 5. Broken session stays in map after `handleRequest` failure

**Severity**: Important (repeated failures for one client)
**Location**: `src/mcp.ts` line 720

If `session.transport.handleRequest()` throws, the outer catch returns 500 but the session stays in the `sessions` map in a potentially broken state. The next request from the same client will find that session and fail again, producing repeated errors with no recovery path until the 5-minute idle timeout.

**Fix**: Catch `handleRequest` errors at the session level, evict the broken session, and return a JSON-RPC error instructing the client to re-initialize.

---

### 6. No cap on session count — unbounded memory growth

**Severity**: Important (resource exhaustion)
**Location**: `src/mcp.ts` lines 699–711

Each new client with no session ID gets a fresh session with no upper bound. A misbehaving client that reconnects on every call creates one session per request. Sessions live for up to 6 minutes (5-min timeout + 60s polling interval) before cleanup.

**Fix**: Add a `MAX_SESSIONS` guard:
```typescript
if (!session) {
  const MAX_SESSIONS = 50;
  if (sessions.size >= MAX_SESSIONS) {
    nodeRes.writeHead(503, { "Content-Type": "application/json" });
    nodeRes.end(JSON.stringify({ error: "Too many active sessions" }));
    return;
  }
  // ... create session
}
```

---

### 7. `describeRequest()` logging broken for `query` tool

**Severity**: Important (misleading logs)
**Location**: `src/mcp.ts` lines 579–583

```typescript
if (args?.query) {
  const q = String(args.query).slice(0, 80);
  return `tools/call ${tool} "${q}"`;
}
```

The `query` tool now accepts a `searches` array, not a top-level `query` string. This condition is never true for query tool calls, so all `tools/call query` requests log without any search text.

**Fix**:
```typescript
if (args?.searches && Array.isArray(args.searches) && args.searches.length > 0) {
  const firstSearch = args.searches[0];
  const q = String(firstSearch?.query ?? "").slice(0, 80);
  return `tools/call ${tool} [${firstSearch?.type}] "${q}"`;
}
if (args?.query) { /* keep for any other tools that still use it */ }
```

---

### 8. `setInterval` not `.unref()`'d — holds event loop open

**Severity**: Important (test infrastructure)
**Location**: `src/mcp.ts` line 603

The session cleanup interval prevents the Node.js event loop from exiting naturally after `stop()` is called (e.g., in test teardown), keeping the process alive for up to 60 seconds.

**Fix**: Add `sessionCleanup.unref()` immediately after creating the interval.

---

## Medium Issues (3 — nice to have)

### 9. Outer catch returns plain-text body on a JSON-RPC endpoint

**Location**: `src/mcp.ts` lines 761–765

The outer catch returns `"Internal Server Error"` as plain text with no `Content-Type` header. MCP clients expecting JSON-RPC will fail with a secondary parse error. Also no `headersSent` guard — if headers were already partially written, `writeHead(500)` throws `ERR_HTTP_HEADERS_SENT`.

### 10. GET/DELETE session-not-found error inconsistent with POST path

**Location**: `src/mcp.ts` lines 731–734

The POST path returns a proper JSON-RPC error with `"Session expired. Please re-initialize."`. The GET/DELETE path returns `{ error: "Session not found" }` — a different format and less actionable message.

### 11. Docs say stale sessions produce "404" but code returns HTTP 400

**Locations**: CLAUDE.md line 204, README.md line 244

Both documents reference `404 "Session not found"` but the code uses HTTP 400 with JSON-RPC error code -32600.

---

## Files to Remove from PR

These files are included in the branch diff but should not go to the upstream repo:

| File | Reason |
|------|--------|
| `docs/session-summary-2026-02-24-mcp-http-migration.md` | Private Claude session summary — internal dev workflow notes, not relevant to upstream users |
| `docs/session-summary-2026-03-01-launchagent-v1.1.0-deploy.md` | Same — internal session continuity notes with personal paths and debugging history |
| `docs/node-101-package-managers.md` | General Node.js ecosystem education article with internal case study. Well-written but not project-specific docs |

**How to remove from PR without deleting locally**: These files should be excluded from the feature branch before the PR. If they're already committed, they can be removed from the branch with:
```bash
git checkout feature/mcp-multi-client
git rm --cached docs/session-summary-2026-02-24-mcp-http-migration.md
git rm --cached docs/session-summary-2026-03-01-launchagent-v1.1.0-deploy.md
git rm --cached docs/node-101-package-managers.md
git commit -m "chore: remove internal docs from PR scope"
```
Note: `--cached` removes from git tracking only; the files remain on disk.

---

## Strengths

- The per-session `Map<string, Session>` architecture is sound — correct separation of concerns between clients
- Session lifecycle is complete: creation, routing by header, idle cleanup, explicit termination via DELETE, graceful shutdown
- README Session Management section is accurate and well-written against the actual code
- `bootstrap`/`bootout` LaunchAgent guidance is consistent across all three instruction docs
- The "reconnect client, not server" operational advice is correct and actionable

---

## Recommended Action Plan

1. **Fix merge conflict markers** in CLAUDE.md and README.md on the feature branch
2. **Address critical code issues** (#2 session creation try/catch, #3 cleanup race condition)
3. **Add error logging** to all `.catch(() => {})` patterns (#4)
4. **Remove 3 internal docs** from the branch (#session summaries, node-101)
5. **Fix medium issues** if time permits (#7 logging, #8 unref, #9-11 consistency)
6. **Rebuild, relink, restart LaunchAgent** after code changes
7. **Re-run review** to verify fixes
