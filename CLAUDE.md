# QMD - Query Markup Documents

Use Bun instead of Node.js (`bun` not `node`, `bun install` not `npm install`).

## Commands

```sh
qmd collection add . --name <n>   # Create/index collection
qmd collection list               # List all collections with details
qmd collection remove <name>      # Remove a collection by name
qmd collection rename <old> <new> # Rename a collection
qmd ls [collection[/path]]        # List collections or files in a collection
qmd context add [path] "text"     # Add context for path (defaults to current dir)
qmd context list                  # List all contexts
qmd context check                 # Check for collections/paths missing context
qmd context rm <path>             # Remove context
qmd get <file>                    # Get document by path or docid (#abc123)
qmd multi-get <pattern>           # Get multiple docs by glob or comma-separated list
qmd status                        # Show index status and collections
qmd update [--pull]               # Re-index all collections (--pull: git pull first)
qmd embed                         # Generate vector embeddings (uses node-llama-cpp)
qmd query <query>                 # Search with query expansion + reranking (recommended)
qmd search <query>                # Full-text keyword search (BM25, no LLM)
qmd vsearch <query>               # Vector similarity search (no reranking)
qmd mcp                           # Start MCP server (stdio transport)
qmd mcp --http [--port N]         # Start MCP server (HTTP, default port 8181)
qmd mcp --http --daemon           # Start as background daemon
qmd mcp stop                      # Stop background MCP daemon
```

## Collection Management

```sh
# List all collections
qmd collection list

# Create a collection with explicit name
qmd collection add ~/Documents/notes --name mynotes --mask '**/*.md'

# Remove a collection
qmd collection remove mynotes

# Rename a collection
qmd collection rename mynotes my-notes

# List all files in a collection
qmd ls mynotes

# List files with a path prefix
qmd ls journals/2025
qmd ls qmd://journals/2025
```

## Context Management

```sh
# Add context to current directory (auto-detects collection)
qmd context add "Description of these files"

# Add context to a specific path
qmd context add /subfolder "Description for subfolder"

# Add global context to all collections (system message)
qmd context add / "Always include this context"

# Add context using virtual paths
qmd context add qmd://journals/ "Context for entire journals collection"
qmd context add qmd://journals/2024 "Journal entries from 2024"

# List all contexts
qmd context list

# Check for collections or paths without context
qmd context check

# Remove context
qmd context rm qmd://journals/2024
qmd context rm /  # Remove global context
```

## Document IDs (docid)

Each document has a unique short ID (docid) - the first 6 characters of its content hash.
Docids are shown in search results as `#abc123` and can be used with `get` and `multi-get`:

```sh
# Search returns docid in results
qmd search "query" --json
# Output: [{"docid": "#abc123", "score": 0.85, "file": "docs/readme.md", ...}]

# Get document by docid
qmd get "#abc123"
qmd get abc123              # Leading # is optional

# Docids also work in multi-get comma-separated lists
qmd multi-get "#abc123, #def456"
```

## Options

```sh
# Search & retrieval
-c, --collection <name>  # Restrict search to a collection (matches pwd suffix)
-n <num>                 # Number of results
--all                    # Return all matches
--min-score <num>        # Minimum score threshold
--full                   # Show full document content
--line-numbers           # Add line numbers to output

# Multi-get specific
-l <num>                 # Maximum lines per file
--max-bytes <num>        # Skip files larger than this (default 10KB)

# Output formats (search and multi-get)
--json, --csv, --md, --xml, --files
```

## Development

```sh
bun src/qmd.ts <command>   # Run from source
bun link               # Install globally as 'qmd'
```

## Tests

All tests live in `test/`. Run everything:

```sh
npx vitest run --reporter=verbose test/
bun test --preload ./src/test-preload.ts test/
```

## Architecture

- SQLite FTS5 for full-text search (BM25)
- sqlite-vec for vector similarity search
- node-llama-cpp for embeddings (embeddinggemma), reranking (qwen3-reranker), and query expansion (Qwen3)
- Reciprocal Rank Fusion (RRF) for combining results
- Smart chunking: 900 tokens/chunk with 15% overlap, prefers markdown headings as boundaries

## Important: Do NOT run automatically

- Never run `qmd collection add`, `qmd embed`, or `qmd update` automatically
- Never modify the SQLite database directly
- Write out example commands for the user to run manually
- Index is stored at `~/.cache/qmd/index.sqlite`

## MCP Server (HTTP mode)

The MCP server runs as a shared HTTP daemon on port **8181**, started automatically at login via macOS LaunchAgent (`~/Library/LaunchAgents/com.qmd.mcp.plist`).

```sh
# Manual control
qmd mcp --http                # foreground (Ctrl-C to stop)
qmd mcp --http --daemon       # background, writes PID to ~/.cache/qmd/mcp.pid
qmd mcp stop                  # stop background daemon

# LaunchAgent control (use bootstrap/bootout, NOT load/unload)
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.qmd.mcp.plist  # start
launchctl bootout gui/$(id -u)/com.qmd.mcp                                 # stop
curl http://localhost:8181/health                                           # check status
```

- **Endpoint**: `POST http://localhost:8181/mcp` (Streamable HTTP, stateless)
- **Health check**: `GET http://localhost:8181/health`
- **Logs**: `/tmp/qmd-mcp.log` and `/tmp/qmd-mcp.error.log`
<<<<<<< HEAD
- **Binary**: `~/.nvm/versions/node/v24.12.0/bin/qmd` (update path if nvm node version changes)
=======
- **Binary**: `~/.bun/bin/qmd` (symlink to local project via `bun link`)

### Health check & status

```sh
curl http://localhost:8181/health    # {"status":"ok","uptime":<seconds>}
qmd status                          # shows "MCP: running (PID ...)"
```

### Killing orphaned processes

If `qmd mcp stop` says "no PID file" but the port is occupied:

```sh
lsof -i :8181                       # find what owns the port
lsof -ti :8181 | xargs kill         # kill it (graceful)
lsof -ti :8181 | xargs kill -9      # force kill
```

If the server keeps restarting after `kill`, the LaunchAgent is re-spawning it — use `launchctl bootout` instead.

### Session errors

If an MCP client gets `404 "Session not found"` or `"Session expired"`, the server restarted and the client has a stale session ID. **Reconnect the client, not the server** — the server is healthy. In Claude Code, run `/mcp` to reconnect.

The REST endpoints (`/query`, `/search`, `/health`) are stateless and unaffected by session issues.

### Reference

- **MCP endpoint**: `POST http://localhost:8181/mcp` (Streamable HTTP, stateful sessions)
- **REST endpoint**: `POST http://localhost:8181/query` (stateless, no MCP envelope)
- **Health check**: `GET http://localhost:8181/health`
>>>>>>> 29209f7 (docs: document multi-client deployment pipeline and fix stale paths)
- Claude Code connects via HTTP through `~/.claude/.mcp.json` (not the plugin marketplace.json, which only supports stdio)
- LLM models stay loaded in VRAM and are shared across all connected agents
- Embedding/reranking contexts auto-dispose after 5 min idle, recreate on next request (~1s)
- Multiple MCP clients (Claude Code, Gemini CLI) can connect simultaneously via per-session architecture

### Deploying source changes to the LaunchAgent

After modifying `src/mcp.ts` or other source files, the LaunchAgent won't pick up changes until you rebuild and relink:

```sh
bun run build                    # compile src/ → dist/
bun link                         # update ~/.bun/bin/qmd symlink
launchctl bootout gui/$(id -u)/com.qmd.mcp
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.qmd.mcp.plist
```

**Common pitfall**: forgetting `bun link` after `bun run build`. The LaunchAgent runs `~/.bun/bin/qmd`, which is a symlink into `~/.bun/install/global/node_modules/@tobilu/qmd/dist/`. Without `bun link`, the global copy stays stale even though `dist/` is updated locally.

## Do NOT compile

- Never run `bun build --compile` - it overwrites the shell wrapper and breaks sqlite-vec
- The `qmd` file is a shell script that runs compiled JS from `dist/` - do not replace it
- `npm run build` compiles TypeScript to `dist/` via `tsc -p tsconfig.build.json`

## Releasing

Use `/release <version>` to cut a release. Full changelog standards,
release workflow, and git hook setup are documented in the
[release skill](skills/release/SKILL.md).

Key points:
- Add changelog entries under `## [Unreleased]` **as you make changes**
- The release script renames `[Unreleased]` → `[X.Y.Z] - date` at release time
- Credit external PRs with `#NNN (thanks @username)`
- GitHub releases roll up the full minor series (e.g. 1.2.0 through 1.2.3)
