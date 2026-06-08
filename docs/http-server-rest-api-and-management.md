---
title: QMD HTTP Server — REST Search API & Server Management
status: salvaged-reference
source_branch: dev-old
source_commit: f5597ba
source_date:   2026-03-11
salvaged_date: 2026-06-07
verified_against: v2.5.3 (dev @ 02e4766)
related_docs:
  - plan-documentation-sprint-2026-06-06-canonical.md
related_pr: 715
tags: [mcp, http, rest-api, launchagent, daemon, session-management, ops]
---

# QMD HTTP Server — REST Search API & Server Management

> **Provenance.** This content originated in the `dev-old` branch's README
> (commit `f5597ba`, 2026-03-11) and was the only markdown content unique to that
> branch — it was deliberately dropped during the v2.0 migration because the
> upstream README had been rewritten (see
> `docs/branch-assessment-2026-03-11-v2-upgrade-plan.md`, "Files intentionally
> NOT copied"). Salvaged on 2026-06-07 and re-verified against current source
> (`src/mcp/server.ts`, `src/llm.ts`, `src/index.ts` at `dev` @ `02e4766`).
> Two claims that had drifted from current behavior were corrected and are
> flagged inline with **[Corrected 2026-06-07]**. Candidate for a future
> documentation-sprint phase / README PR.

## HTTP Endpoints

The HTTP server exposes the following endpoints:

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/mcp` | `POST`, `GET`, `DELETE` | MCP Streamable HTTP (JSON-RPC, stateful sessions, used by MCP clients). `POST initialize` creates a session; later requests must include `mcp-session-id`. |
| `/health` | `GET` | Liveness check — returns `{"status":"ok","uptime":<seconds>}` |
| `/query` | `POST` | REST search API (no MCP envelope needed) |
| `/search` | `POST` | Alias for `/query` |

> **[Corrected 2026-06-07]** After ~5 minutes of inactivity the server disposes
> idle LLM resources to free VRAM and transparently recreates them on the next
> request (~1s penalty). In the current SDK this disposes the **models too**, not
> just the embedding/reranking contexts (`src/index.ts` sets
> `inactivityTimeoutMs: 5 * 60 * 1000` with `disposeModelsOnInactivity: true`;
> the timer lives in `src/llm.ts`). Unloading is gated by `canUnloadLLM()` —
> active sessions/operations block it and reschedule the timer. *(The original
> README claimed only contexts were disposed and "models remain loaded"; that is
> no longer the default behavior.)*

Point any MCP client at `http://localhost:8181/mcp` to connect.

## REST Search API

The `/query` (and `/search`) endpoint lets you search QMD directly over HTTP without an MCP client. This is useful for scripting, monitoring, or integrating with non-MCP tools.

```sh
# Keyword search
curl -s http://localhost:8181/query \
  -H "Content-Type: application/json" \
  -d '{"searches": [{"type": "lex", "query": "rate limiter"}], "limit": 5}'

# Semantic search
curl -s http://localhost:8181/query \
  -H "Content-Type: application/json" \
  -d '{"searches": [{"type": "vec", "query": "how does authentication work"}]}'

# Combined (lex + vec) with collection filter
curl -s http://localhost:8181/query \
  -H "Content-Type: application/json" \
  -d '{
    "searches": [
      {"type": "lex", "query": "\"connection pool\" timeout"},
      {"type": "vec", "query": "why do database connections time out"}
    ],
    "collections": ["docs"],
    "limit": 10,
    "minScore": 0.3
  }'
```

**Request body:**

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `searches` | array | *(required)* | Array of `{type, query}` objects. Types: `lex`, `vec`, `hyde` |
| `collections` | string[] | default-included collections | Restrict to specific collections. If omitted, the server uses the configured default-included collections; if that list is empty, search falls back to all collections. |
| `limit` | number | 10 | Max results to return |
| `minScore` | number | 0 | Minimum relevance threshold (0.0–1.0) |
| `candidateLimit` | number | store default | Max candidates to rerank |
| `intent` | string | *(none)* | Disambiguation context for query expansion, reranking, and snippet extraction |
| `rerank` | boolean | store default | Enable or disable LLM reranking |

**Response:** `{"results": [{docid, file, title, score, context, line, snippet}, ...]}`

## MCP Server Management

There are two ways to run the HTTP server long-term: **daemon mode** (`--daemon` flag) and **macOS LaunchAgent**. They are mutually exclusive — use one or the other.

| | Daemon mode | LaunchAgent |
|---|---|---|
| **Start** | `qmd mcp --http --daemon` | `launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.qmd.mcp.plist` |
| **Stop** | `qmd mcp stop` | `launchctl bootout gui/$(id -u)/com.qmd.mcp` |
| **Auto-restart on crash** | No | Yes (`KeepAlive: true`) |
| **Starts at login** | No | Yes (`RunAtLoad: true`) |
| **PID file** | `~/.cache/qmd/mcp.pid` | None (launchd manages the process) |
| **Logs** | `~/.cache/qmd/mcp.log` | `/tmp/qmd-mcp.log` and `/tmp/qmd-mcp.error.log` |

> **Important**: `qmd mcp stop` only works for daemon mode (it reads the PID file). LaunchAgent-managed processes must use `launchctl bootout`. *Mixing these up is a common source of orphaned processes.*

### Restarting

```sh
# Daemon mode
qmd mcp stop && qmd mcp --http --daemon

# LaunchAgent
launchctl bootout gui/$(id -u)/com.qmd.mcp  # stops
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.qmd.mcp.plist  # starts
```

### Health Check

```sh
curl http://localhost:8181/health
# Returns: {"status":"ok","uptime":<seconds>}

qmd status
# Daemon mode only: shows MCP: running (PID ...) by checking the PID file
```

### Killing Orphaned Processes

If `qmd mcp stop` reports "no PID file" but port 8181 is still occupied (e.g., after a crash, or when the LaunchAgent was managing the process):

```sh
# Find what's using port 8181
lsof -i :8181

# Kill by PID (graceful)
kill <pid>

# Force-kill everything on the port
lsof -ti :8181 | xargs kill -9
```

### Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `Port 8181 already in use` | Another process owns the port | `lsof -ti :8181 \| xargs kill` then restart |
| `qmd mcp stop` says "not running" but server is up | LaunchAgent is managing the process (no PID file) | `launchctl bootout gui/$(id -u)/com.qmd.mcp` |
| `Cleaned up stale PID file` | Previous daemon crashed without cleanup | Normal — PID file was removed, safe to restart |
| Server keeps restarting after `kill` | LaunchAgent `KeepAlive` is re-spawning it | Use `launchctl bootout` instead of `kill` |
| `Session not found` / `Session expired` from MCP client | Server restarted and client has a stale session ID | Reconnect the **client**, not the server (e.g., `/mcp` in Claude Code). The server is healthy — see [Session Management](#session-management) |
| Slow first query after restart or idle unload | Models and contexts may be reloading into memory/VRAM | Expected; the server recreates resources on demand |

### Graceful Shutdown Behavior

When the server receives SIGTERM or SIGINT, it:
1. Closes all active MCP session transports (stops accepting new requests)
2. Closes the HTTP server
3. Closes the SQLite database connection
4. Disposes LLM models from VRAM
5. Exits

### Session Management

The MCP `/mcp` endpoint uses **stateful sessions** with per-client isolation. Each connecting client (Claude Code, Gemini CLI, etc.) gets its own `McpServer` + transport pair, keyed by `mcp-session-id` header. Multiple clients can connect simultaneously — they share the underlying search index and the server's lazily loaded LLM resources.

When a client connects, it sends an `initialize` request and receives a session ID. All subsequent requests must include this session ID.

> **[Corrected 2026-06-07]** Sessions persist until the client's transport closes
> (`onsessionclosed` → `sessions.delete`) or the server restarts — there is **no
> idle-session timeout**. Current `src/mcp/server.ts` registers no timer for
> session expiry (verified: zero `setInterval`/`setTimeout` in the file). *(The
> original README claimed "idle sessions are cleaned up automatically after 5
> minutes" — that 5-minute timer is the LLM resource-unload timer in
> `src/llm.ts`, which is unrelated to session lifetime.)*

If the server restarts, all in-memory session IDs are lost. Clients that send the old session ID will receive a `404 "Session not found"` error. **The fix is to reconnect the client, not restart the server** — the server is already running fine with a clean state.

| Client | How to reconnect |
|--------|-----------------|
| Claude Code | Run `/mcp` to reconnect MCP servers |
| Gemini CLI | Restart the Gemini CLI session |
| Other MCP clients | Re-send an `initialize` request (client-specific) |

> **Note**: The REST endpoints (`/query`, `/search`, `/health`) are truly stateless and are not affected by session issues. If you need to verify the server is healthy, use `curl http://localhost:8181/health`.
