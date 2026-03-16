# QMD Local Environment -- OVERRIDES CLAUDE.md

**This file is authoritative for our local development environment.**
**Where this file contradicts CLAUDE.md, THIS FILE WINS.** CLAUDE.md is the upstream
project's general-purpose instructions and may contain recommendations (like using Bun)
that are WRONG for our setup. Do not follow CLAUDE.md blindly -- always check here first
for runtime, build, install, and daemon management instructions.

## Current Install State

> **UPDATE THIS SECTION whenever the install method, version, or binary path changes.**

QMD is installed **from local source code**, NOT from the published npm package on npmjs.

- **Source**: `/Users/rymalia/projects/qmd` (git clone of `tobi/qmd` fork)
- **Binary**: `/Users/rymalia/.nvm/versions/node/v24.12.0/bin/qmd` (via `npm link`)
- **Symlink chain**: `~/.nvm/.../bin/qmd` -> `node_modules/@tobilu/qmd` -> `/Users/rymalia/projects/qmd`
- **Runtime**: Node.js v24.12.0 (via nvm)
- **Version**: v2.0.1 (source build on `dev` branch)

Any change to source followed by `npm run build` is immediately live. No reinstall or re-link needed after the initial `npm link`.

### Update/Reinstall from Source

```sh
launchctl unload ~/Library/LaunchAgents/com.qmd.mcp.plist
git pull
npm run build && npm link
launchctl load ~/Library/LaunchAgents/com.qmd.mcp.plist
```

### Quick Rebuild (no pull, just pick up local changes)

```sh
npm run build
pkill -f "qmd.js mcp --http"   # LaunchAgent auto-restarts with new code
```

## OVERRIDE: Runtime is Node.js, NOT Bun

**CLAUDE.md says "Use Bun." IGNORE THAT. Do NOT use Bun** for anything in this project.
This is an intentional, permanent decision made during the v2.0 upgrade (2026-03-11).

**Why:** Bun's built-in SQLite (`bun:sqlite`) does not support `loadExtension()`, which
means sqlite-vec cannot load. This silently breaks vector search, hybrid queries, and
`qmd cleanup`. The `bin/qmd` wrapper script detects the runtime by checking for `bun.lock`
-- if it exists, the wrapper routes to Bun, causing ABI mismatches when deps were compiled
for Node.

**Rules:**
- Never run `bun install` -- it recreates `bun.lock`, which flips the runtime back to Bun
- Never run `bun test`, `bun run`, `bun link`, or any `bun` command
- Use `npm install`, `npm run build`, `npm link` for all operations
- Use `npx vitest` for testing
- The `bun.lock` file was intentionally deleted from this checkout -- do not recreate it

**CLAUDE.md references `bun src/cli/qmd.ts`, `bun link`, `bun test` — replace all of
those with their npm equivalents when following any instructions from that file.**

## OVERRIDE: MCP Server -- LaunchAgent Managed

**CLAUDE.md lists `qmd mcp stop` as a command. Do NOT use it in our environment.**
The QMD MCP HTTP server runs as a macOS LaunchAgent, NOT via `qmd mcp --daemon`.

- **Plist**: `~/Library/LaunchAgents/com.qmd.mcp.plist`
- **Label**: `com.qmd.mcp`
- **Command**: `qmd mcp --http` (port 8181)
- **KeepAlive**: true (auto-restarts on crash or kill)
- **RunAtLoad**: true (starts on login)
- **Logs**: `/tmp/qmd-mcp.log` (stdout), `/tmp/qmd-mcp.error.log` (stderr)
- **Log timezone**: UTC, not local time. User is in PDT (UTC-7).

### Stop/Start

```sh
# Stop (prevents auto-restart)
launchctl unload ~/Library/LaunchAgents/com.qmd.mcp.plist

# Start
launchctl load ~/Library/LaunchAgents/com.qmd.mcp.plist
```

### What NOT to use

- `qmd mcp stop` -- only works for `--daemon` mode (PID files at `~/.cache/qmd/mcp.pid`). Our LaunchAgent doesn't use `--daemon`, so there is no PID file.
- `kill <pid>` -- LaunchAgent will restart the process immediately (`KeepAlive: true`).

### After Rebuilding

After `npm run build`, kill the process and LaunchAgent restarts it with the new code within seconds:

```sh
pkill -f "qmd.js mcp --http"
```

Or for a clean stop/start: `launchctl unload` then `launchctl load`.

## Three-Layer Integration Architecture

QMD integrates with Claude Code via three independent systems:

### Layer 1: MCP Connection (`~/.claude.json`)

```json
{ "mcpServers": { "qmd": { "type": "http", "url": "http://localhost:8181/mcp" } } }
```

Connects to the LaunchAgent daemon. Provides the actual tools: `query`, `get`, `multi_get`, `status`.

### Layer 2: Plugin (`~/.claude/plugins/`)

Provides the skill file (SKILL.md) that teaches Claude how to write good QMD queries. Re-clones from GitHub on session start. Plugin version `0.1.0` is the plugin *format* version, not the QMD software version.

Currently pointing to local repo (`git:///Users/rymalia/projects/qmd` in `known_marketplaces.json`) for dev iteration. Revert to `github:tobi/qmd` after upstream PRs merge.

### Layer 3: `qmd skill install` (optional, redundant for us)

Standalone copy of the same SKILL.md. Not needed -- the plugin already provides it.

### Key points

- Plugin provides the *knowledge* (how to query). MCP provides the *tools* (the query function).
- They're in different config files, managed by different systems.
- If the daemon is down, MCP tools disappear but the skill still loads.
- After upgrading QMD, restart the daemon. The plugin auto-updates from GitHub.

## v2.0 Architecture

Source split into `src/cli/` + `src/mcp/` + SDK layer (changed from v1.x single-directory layout):

```
src/index.ts          SDK entry point -- public QMDStore interface, createStore()
  src/store.ts        Internal data layer (SQLite, FTS5, vectors, search pipeline)
  src/maintenance.ts  Cleanup, indexing operations

src/cli/qmd.ts        CLI -- consumes SDK
src/mcp/server.ts     MCP server -- consumes SDK
src/embedded-skills.ts  Powers `qmd skill install`
```

MCP tools go through the `QMDStore` interface, not raw store methods.

### Database

SQLite at `~/.cache/qmd/index.sqlite` with WAL mode:

| Table | Purpose |
|-------|---------|
| **content** | Content-addressable store (hash -> full text) |
| **documents** | Virtual path layer (collection + path -> hash), `active` flag for soft deletes |
| **documents_fts** | FTS5 virtual table (porter unicode61 tokenizer) |
| **content_vectors** | Embedding chunk metadata (hash, sequence, position) |
| **vectors_vec** | sqlite-vec virtual table (float[384], cosine distance) |
| **llm_cache** | Cached LLM responses keyed by input hash |

### Search Pipeline (hybrid `query` command)

1. **BM25 probe** -- quick FTS check for strong signal (score >= 0.85, gap >= 0.15 from #2)
2. **Query expansion** -- LLM generates variations (skipped if strong signal found)
3. **Type-routed search** -- each expanded query runs FTS ('lex') or vector ('vec'/'hyde')
4. **RRF fusion** -- k=60, original query gets 2x weight, top-rank bonus
5. **Chunk selection** -- best chunk selected by keyword overlap
6. **LLM reranking** -- top 40 candidates via qwen3-reranker (yes/no + logprobs)
7. **Position-aware blending** -- ranks 1-3: 75% RRF / 25% reranker; 4-10: 60/40; 11+: 40/60

## Agent Preferences

- **Subagent model**: Always use `sonnet` for Task tool subagents, never `haiku`
