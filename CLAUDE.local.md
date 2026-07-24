# QMD Local Environment -- OVERRIDES CLAUDE.md

> **TL;DR — Install method:** `qmd` runs from **local source** (`/Users/rymalia/projects/qmd`) linked via `npm link`, **NOT** the published `@tobilu/qmd` npm package. Edits go live after `npm run build`. Details in [Current Install State](#current-install-state).

**This file is authoritative for our local development environment.**
**Where this file contradicts CLAUDE.md, THIS FILE WINS.** CLAUDE.md is the upstream
project's general-purpose instructions and may contain recommendations that conflict
with our specific setup (LaunchAgent-managed daemon, source-mode local install,
npm-primary toolchain). Check here first for runtime, build, install, and daemon
management instructions.

## Current Install State

> **UPDATE THIS SECTION whenever the install method, version, or binary path changes.**

QMD is installed **from local source code**, NOT from the published npm package on npmjs.

- **Source**: `/Users/rymalia/projects/qmd` (git clone of `tobi/qmd` fork)
- **Binary**: `/Users/rymalia/.nvm/versions/node/v24.18.0/bin/qmd` (via `npm link`)
- **Symlink chain**: `~/.nvm/.../bin/qmd` -> `node_modules/@tobilu/qmd` -> `/Users/rymalia/projects/qmd`
- **Runtime**: Node.js v24.18.0 (via nvm; upgraded from v24.12.0 ~2026-06-30 — the old nvm dir was removed, which broke the LaunchAgent's hardcoded path until it was repointed on 2026-07-07)
- **Version**: v2.6.3 (source, `dev` branch, commit d39452d — merge of main/e428df7 into dev on 2026-07-24). **`dev` is the working branch**: it equals main plus 25 local docs commits (session summaries, doc-sprint artifacts, this file) with **zero source divergence** — `git diff main dev -- src/ bin/ test/ package.json` is empty. Note v2.6.3 is upstream main's *unpublished* version: release PR #746 merged 2026-06-24 but was never tagged/published, so npm and GitHub releases still say v2.5.3.

Any change to source followed by `npm run build` is immediately live. No reinstall or re-link needed after the initial `npm link`.

### Determining the current version (REQUIRED procedure)

> **When asked "what version are we running?" — do NOT trust a single source.**
> Because this is a source-mode `npm link` install, the running version, the
> source, and the compiled `dist/` can disagree. A bare `git pull` updates
> `package.json` + `src/` and goes live instantly (the launcher runs `src/` via
> tsx), while `dist/` stays at whatever was last built. Always cross-check all
> three:

```sh
qmd --version                         # what's actually running (reads package.json + git HEAD)
grep '"version"' package.json | head -1   # source version
git rev-parse --short HEAD             # source commit (matches the (hash) in --version)
stat -f "%Sm  %N" -t "%Y-%m-%d %H:%M" dist/cli/qmd.js src/cli/qmd.ts  # build freshness
```

- If `dist/cli/qmd.js` is **older** than `src/cli/qmd.ts`, `dist/` is stale.
  It doesn't affect the running version *today* (the launcher prefers `src/`
  via tsx), BUT it's the fallback if tsx is ever unavailable — a stale `dist/`
  means a silent downgrade to old code. Flag it and offer `npm run build`.
- The `(hash)` in `qmd --version` is read live from git HEAD; if it doesn't
  match `git rev-parse --short HEAD`, something is running from a different
  checkout. Investigate before trusting the version number.
- **The running daemon's reported version can lie.** The MCP `serverInfo.version`
  (and `qmd --version`) read `package.json` live at request time — a daemon
  started before a branch switch/rebuild reports the NEW version while running
  OLD code from memory (observed 2026-07-07: daemon started on dev/v2.5.3
  reported "2.6.3" after the checkout moved to main). The truth is the process
  start time: `ps -o lstart -p $(pgrep -f "cli/qmd.ts mcp --http")` — if it
  predates the rebuild/switch, restart the daemon.

### Update/Reinstall from Source

```sh
launchctl unload ~/Library/LaunchAgents/com.qmd.mcp.plist
git pull                                   # syncs origin/dev (our docs) only
git fetch upstream && git merge upstream/main   # pulls new upstream code into dev
npm run build && npm link
launchctl load ~/Library/LaunchAgents/com.qmd.mcp.plist
```

### Quick Rebuild (pick up local source changes)

```sh
npm run build
pkill -f "cli/qmd.[tj]s mcp --http"   # LaunchAgent auto-restarts with new code
```

(The old documented pattern `pkill -f "qmd.js mcp --http"` matches nothing in
source mode — the process cmdline is `.../src/cli/qmd.ts mcp --http` under tsx.
The pattern above matches both source and dist mode. Verified 2026-07-07.)

**Source mode note (re-verified on v2.6.3):** In a git checkout, the `bin/qmd`
launcher (now a cross-platform Node script, no longer POSIX shell) runs
`src/cli/qmd.ts` instead of `dist/cli/qmd.js`. Source-mode detection gates on the
package dir NOT being inside `node_modules/` (override: `QMD_SOURCE_MODE=1`/`0`),
and the runner is **lockfile-driven**: `package-lock.json` present → Node+tsx
(wins over `bun.lock` even when both exist); only `bun.lock` + bun on PATH → Bun.
Source changes take effect on the next daemon restart even without `npm run build`.
The build is still recommended for consistency (dist/ matches src/) and for
catching TypeScript errors before runtime.

## Runtime: Node Daemon, npm Primary, Bun Tolerated

**Status as of v2.6.3 (re-verified 2026-07-07):** `doctor` fully clean under
better-sqlite3 (SQLite 3.53.1, sqlite-vec v0.1.9, GPU Metal probe on M3, 3,486
docs on fingerprint c37385, vector sample reproduces). `search`, `vsearch`,
`query` (incl. multi-line typed query documents with `intent:` — new in 2.6.x,
see docs/SYNTAX.md), `get`, `multi-get`, and all four MCP tools ran live.
`embed` was NOT re-run (banned from auto-execution), but doctor's
embedding-freshness + vector-sample checks validate the embed output is intact.
Upstream still maintains dual-runtime support (`npm test` runs the Bun suite).
New in 2.6.x: the launcher sets `GGML_METAL_NO_RESIDENCY=1` on darwin (avoids a
llama.cpp Metal teardown abort; doctor reports it as an expected env override).

**Current setup:**

- **MCP daemon (the thing Claude Code talks to):** Node.js v24.18.0 via tsx.
  Runner selection is now lockfile-driven (see Source mode note above): our
  untracked local `package-lock.json` (from `npm install`) forces Node+tsx
  regardless of PATH. This is the well-tested, stable runtime for the
  long-running process -- keep it that way.
- **Interactive `qmd` from your shell:** Also Node+tsx now, always — v2.6.x's
  launcher prefers `package-lock.json` over `bun.lock`, so bun on PATH no
  longer flips the runtime (pre-2.6 behavior where Bun could be selected is
  obsolete; verified 2026-07-07, doctor reports `better-sqlite3`). To get Bun
  you'd have to delete `package-lock.json` — don't.
- **`bun.lock` is tracked upstream.** Do not delete it locally -- it will come
  back via rebases. `package-lock.json` is local-only (untracked) and is what
  pins us to Node.

**Rules:**

- Use `npm install`, `npm run build`, `npm link` for builds -- keeps deps
  consistent with the daemon's runtime.
- `npx vitest` for the local test suite (or `npm test`, which under v2.6.x runs
  `scripts/test-all.mjs`: TypeScript typecheck + vitest under Node + `bun test`
  + package smoke test).
- **Do NOT add `bun` to the LaunchAgent's PATH.** (Since v2.6.x the lockfile
  decides the runtime, so bun on PATH alone can't flip the daemon anymore --
  but keep PATH minimal anyway; the rule costs nothing.)
- `bun install` is still suspect -- it rewrites `bun.lock` from `package.json`
  rather than respecting upstream's pinned versions, which can produce ABI
  mismatches for node-llama-cpp's native binaries. If you need to refresh deps,
  prefer `npm install`.

**Sanity check after any major change:** Run `qmd doctor`. It reports the
active runtime, verifies sqlite-vec, embedding fingerprints, GPU probe, and
content-hash sampling. In our setup the runtime line should always read
`better-sqlite3` (Node) — `bun:sqlite` would mean `package-lock.json` went
missing. If it reports something else, investigate before heavy ops.

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
pkill -f "cli/qmd.[tj]s mcp --http"
```

Or for a clean stop/start: `launchctl unload` then `launchctl load`.

**Then re-verify what's actually loaded** (the daemon's reported version alone
is not proof — see "Determining the current version"): the process start time
from `ps -o lstart -p $(pgrep -f "cli/qmd.ts mcp --http")` must postdate the
rebuild.

## Three-Layer Integration Architecture

QMD integrates with Claude Code via three independent systems:

### Layer 1: MCP Connection (`~/.claude.json`)

```json
{ "mcpServers": { "qmd": { "type": "http", "url": "http://localhost:8181/mcp" } } }
```

Connects to the LaunchAgent daemon. Provides the actual tools: `query`, `get`, `multi_get`, `status`.

### Layer 2: Plugin (`~/.claude/plugins/`)

Provides the skill file (SKILL.md) that teaches Claude how to write good QMD queries. Re-clones from GitHub on session start. Plugin version `0.1.0` is the plugin *format* version, not the QMD software version.

Currently pointing to local repo (`/Users/rymalia/projects/qmd` in `known_marketplaces.json`) for dev iteration. The original reason to wait — upstream docs PRs #718/#719 — resolved: both MERGED as of 2026-07-07 (#716 is an open *issue*, not a PR). Reverting to `github:tobi/qmd` is now unblocked whenever we're done iterating locally. The PR-watch cloud cron routine watching those PRs can be disabled.

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

src/cli/qmd.ts        CLI -- consumes SDK (also reads repo skills/ dir for `qmd skill install`)
src/mcp/server.ts     MCP server -- consumes SDK
```

(`src/embedded-skills.ts` no longer exists in v2.6.x — skill content lives in
the repo's `skills/` directory, read by the CLI.)

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
| **store_collections** | Collection registry (new in 2.6.x; config source of truth remains `~/.config/qmd/index.yml`) |

### Search Pipeline (hybrid `query` command)

1. **BM25 probe** -- quick FTS check for strong signal (score >= 0.85, gap >= 0.15 from #2)
2. **Query expansion** -- LLM generates variations (skipped if strong signal found)
3. **Type-routed search** -- each expanded query runs FTS ('lex') or vector ('vec'/'hyde')
4. **RRF fusion** -- k=60, original query gets 2x weight, top-rank bonus
5. **Chunk selection** -- best chunk selected by keyword overlap
6. **LLM reranking** -- top 40 candidates via qwen3-reranker (yes/no + logprobs)
7. **Position-aware blending** -- ranks 1-3: 75% RRF / 25% reranker; 4-10: 60/40; 11+: 40/60

(All constants re-verified against v2.6.3 source on 2026-07-07: strong-signal
probe 0.85 / gap 0.15, `RERANK_CANDIDATE_LIMIT = 40`, RRF `k = 60`, blend
weights 0.75/0.60/0.40. New in 2.6.x: the CLI accepts multi-line typed query
documents — `qmd query $'intent: ...\nlex: ...\nvec: ...'` — see docs/SYNTAX.md.)

### Known doc-vs-code gaps: multi-get comma-lists (v2.6.3, verified 2026-07-07)

The CLI and MCP comma-list branches are **two independent implementations with
divergent semantics** (CLI: `src/cli/qmd.ts` `multiGet`, matches bare `d.path`;
MCP/SDK: `src/store.ts` `findDocuments`, matches the reconstructed
`qmd://collection/path` URI). Verified matrix:

| Pattern form | CLI comma-list | MCP `multi_get` |
|---|---|---|
| `docs/SYNTAX.md` (collection-relative) | ✅ | ✅ |
| `qmd/docs/SYNTAX.md` (collection-prefixed) | ❌ "File not found" | ✅ |
| `qmd://qmd/docs/SYNTAX.md` (full URI) | ✅ | ✅ |
| `#abc123` (docid) | ❌ | ❌ |
| `NTAX.md` (filename fragment) | ⚠️ silently matches SYNTAX.md | ⚠️ same |

- **Docids fail everywhere in multi-get** despite CLAUDE.md and README
  documenting `qmd multi-get "#abc123, #def456"`. Neither implementation has a
  docid branch (`get` uses a separate `findDocumentByDocid` that multi-get never
  touches). Single-docid patterns hit the glob branch and fail too.
- **Suffix matching is an unanchored `LIKE '%name'`** — no path-separator
  anchor, so a typo'd name can silently fetch the *wrong document* instead of
  erroring (and the did-you-mean path never fires).
- **Unprefixed names resolve globally with `LIMIT 1`, no `ORDER BY`** — which
  collection wins is arbitrary (`README.md` → `tweet/README.md` here).
- **Safe form: always use full `qmd://collection/path` URIs in comma-lists** —
  the only pattern form that works in every branch of both implementations.
  For docids, fall back to one `get` per docid.
- Related parseability defect: multi-get's `--format files` prepends the docid
  *inside* the first CSV field (`#1b5968 docs/SYNTAX.md,"context"`), so naive
  comma-splitting mangles the first column. (`--format files` itself emitting
  headerless `docid,score,path,"context"` CSV is by design and original
  behavior — shipped in v0.9.0, the first published release.)

Upstream status (2026-07-07): the docid gap was already filed as **#706** with a
fix in review (**PR #753**, covers CLI + SDK + MCP with tests — if merged, the
docid rows above flip to ✅). The path-matching divergence / unanchored-suffix /
arbitrary-LIMIT-1 semantics are NOT covered by #753 (explicit non-goal) — we
filed those as **#759**, and the `--format files` docid-in-first-field defect as
**#760**. We also posted the matrix to the #706 thread and a scope-confirming
comment on #753 (2026-07-07).

## Agent Preferences

- **Subagent model**: Always use `sonnet` for Task tool subagents, never `haiku`
