# Session Summary: QMD v2.0 Upgrade

**Date:** 2026-03-11
**Branch:** `dev` (rebuilt from updated `main` at v2.0.1)
**Previous version:** v1.1.0
**Current version:** v2.0.1

---

## What We Did

Upgraded our QMD fork from v1.1.0 to v2.0.1 (61 upstream commits), preserving local work and documenting everything for future reference.

### Changes Made

| Change | Detail |
|--------|--------|
| **Branch assessment doc** | Created `docs/branch-assessment-2026-03-11-v2-upgrade-plan.md` — comprehensive analysis of all branches, upstream changes, and phased upgrade plan. Reviewed and refined through two rounds of expert feedback |
| **Safety tags** | Created `archive/dev-pre-v2-merge` and `archive/feature-mcp-multi-client` tags (pushed to remote) to preserve old branch history |
| **Updated main** | Fast-forwarded `main` from v1.1.0 to v2.0.1 via `git merge upstream/main` — 61 commits, 5,363 lines added across 31 files |
| **Rebuilt dev** | Created fresh `dev` from updated `main`, selectively restored local docs (13 files: GEMINI.md, session summaries, reference docs, assessment doc) |
| **Dropped superseded work** | Multi-client MCP (`756c89e`), zod pin, reward.py simplifications, README extensions — all superseded by upstream |
| **Installed v2.0.1 (initial)** | `bun install` + `bun run build` + `bun link` — clean install with better-sqlite3@12.6.2, node-llama-cpp@3.17.1, pre-push hook auto-installed |
| **Stopped LaunchAgent** | Bootout before install; not yet re-bootstrapped pending Phase 4c assessment |
| **Hid CLAUDE.local.md** | Renamed to `.bak` to prevent stale v1.1.0 instructions from loading during the transition |
| **Switched runtime from Bun to Node** | `qmd cleanup` crashed with "no such module: vec0" — Bun's built-in SQLite doesn't support `loadExtension()`. Deleted `bun.lock`, reinstalled via `npm install`, rebuilt, and relinked via `npm link`. Binary now at `/Users/rymalia/.nvm/versions/node/v24.12.0/bin/qmd` running under Node with full sqlite-vec support |
| **Removed stale bun symlink** | Deleted `~/.bun/bin/qmd` so `which qmd` resolves to the npm-linked Node binary instead |
| **Pinned zod to 4.2.1 (again)** | `npm audit fix` bumped zod to 4.3.6, causing the same TypeScript build errors we fixed before. `npm install zod@4.2.1 --save-exact` (no caret) resolves it. This is an upstream `package.json` bug — `^4.2.1` is too loose for the MCP SDK's Zod types |
| **Fixed npm audit vulnerabilities** | 6 vulnerabilities (5 high, 1 critical) in hono, express-rate-limit, rollup, simple-git, tar — all resolved via `npm audit fix` patch bumps |
| **Cleaned orphaned vectors** | `qmd cleanup` successfully removed 78 orphaned embedding chunks — first time this command has ever worked, confirming sqlite-vec now loads properly |
| **Tested v2.0 features** | Verified `--explain` score traces, `--intent` disambiguation, and cross-collection search all working |

### Key Decisions

| Decision | Rationale |
|----------|-----------|
| **Drop multi-client feature** | Upstream PR #286 by @joelev shipped the same fix independently. Our idle timeout (5-min TTL) is the only unique addition — worth a separate follow-up PR |
| **Drop reward.py changes** | Upstream's `4511b9b` went the opposite direction (tightened entity detection, added filler penalty vs our simplifications) |
| **File copy instead of cherry-pick** for Phase 3 | The 8 dev commits interleaved kept and dropped content — cherry-picking would require manual unstaging per commit, more error-prone than direct file copy |
| **Phase 3b deferred to after install** | CLAUDE.md cleanup needs knowledge of v2.0's actual structure to make good decisions about what local content to keep |
| **LaunchAgent not yet restarted** | Want to read v2.0's MCP source and test manually before re-enabling auto-start |
| **Switch to Node/npm for runtime** | Bun's SQLite lacks `loadExtension()` support, making sqlite-vec (and therefore vector search, cleanup, and hybrid queries) silently broken. This was a known pre-existing issue (documented in Feb 25 session, audit doc line 148) that was never resolved. v2.0's `bin/qmd` wrapper detects bun vs node by checking for `bun.lock` — removing it and installing via npm makes the wrapper pick node, where better-sqlite3 supports extensions natively |
| **Zod pin is not optional** | The `^4.2.1` range in upstream's `package.json` allows npm to resolve zod 4.3.6, which has breaking type changes against `@modelcontextprotocol/sdk`. Must use `--save-exact`. This only affects source builds (not published npm installs which ship pre-compiled `dist/`). Worth filing upstream |

---

## Current State

### Branch topology

```
main          → ae3604c (v2.0.1, synced with upstream)
dev           → 6e92f09 (1 commit ahead of main: restored local docs)
dev-old       → 29209f7 (old dev, preserved locally)
feature/mcp-multi-client → still exists, tagged as archive
```

### Tags (local + remote)

```
archive/dev-pre-v2-merge          → 29209f7 (old dev tip)
archive/feature-mcp-multi-client  → tip of feature branch
```

### Working tree

```
Modified:  package.json (zod pinned to exact 4.2.1)
           package-lock.json (generated by npm install — replaces bun.lock which was deleted)
           docs/branch-assessment-2026-03-11-v2-upgrade-plan.md (ongoing edits)
           docs/session-summary-2026-03-11-v2-upgrade.md (this file)
Deleted:   bun.lock (removed to make bin wrapper pick node over bun)
Backup:    CLAUDE.local.md.bak
Untracked: finetune/ files, misc docs (feature-plan, llm-architecture, overview)
```

### Runtime

- **Binary**: `/Users/rymalia/.nvm/versions/node/v24.12.0/bin/qmd` (npm link, runs under Node)
- **Old symlink**: `~/.bun/bin/qmd` deleted
- **sqlite-vec**: Working — `qmd cleanup` confirmed vec0 module loads successfully
- **Node version**: v24.12.0 (sufficient; better-sqlite3 ^12.4.5 requires Node 22+)

### LaunchAgent

- **Status**: Stopped (bootout completed, port 8181 confirmed free)
- **Not yet restarted** — pending Phase 4c assessment of v2.0 MCP source
- **Plist note**: The plist PATH includes `/Users/rymalia/.nvm/versions/node/v24.12.0/bin` which is now the correct runtime path (previously a concern, now aligned)

---

## Remaining Work (Phases 4c, 5, 6)

### Phase 4c: Assess MCP and LaunchAgent

Before re-enabling the LaunchAgent:

1. **Read `src/mcp/server.ts`** — understand v2.0's session management. Key questions:
   - Does v2.0's MCP server still use the per-session Map pattern from PR #286?
   - How does the SDK consumer model change the server lifecycle?
   - Any new daemon management features that affect the LaunchAgent setup?

2. **Test manually**: `qmd mcp --http --port 8181` and verify:
   - Health check: `curl http://localhost:8181/health`
   - MCP protocol: connect via Claude Code `/mcp` reconnect
   - Multi-client: confirm concurrent sessions work (this should be built-in now)

3. **Check plist compatibility**:
   - The plist has a hardcoded Node path in PATH env (`/Users/rymalia/.nvm/versions/node/v24.12.0/bin`)
   - ~~v2.0 has a runtime-aware bin wrapper — Bun intercepts first in our setup~~ **RESOLVED**: We switched to Node/npm. The bin wrapper now picks `node` (no `bun.lock` present), and the plist PATH already points to the correct nvm node version
   - The global binary is now at `/Users/rymalia/.nvm/versions/node/v24.12.0/bin/qmd` — **same directory the plist already has in PATH**, so LaunchAgent should work without plist changes

4. **Re-bootstrap** once confirmed:
   ```bash
   launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.qmd.mcp.plist
   ```

### Phase 5: CLAUDE.md / CLAUDE.local.md Deduplication

See `docs/branch-assessment-2026-03-11-v2-upgrade-plan.md` Phase 5 for full plan. Key points:

- Start from upstream's v2.0 `CLAUDE.md` (already on dev)
- Restore `CLAUDE.local.md` from `.bak`
- Compare side-by-side: strip duplicates from `.local.md`, keep only personal preferences and operational knowledge
- Add back any local operational content to `CLAUDE.md` only if still accurate for v2.0
- Goal: zero overlap, clear separation of shared vs private instructions

**What to watch for**: The v2.0 CLAUDE.md references `src/cli/qmd.ts` (not `src/qmd.ts`) and new commands like `qmd skill install`. Our `.local.md` has architecture sections that are now stale — the 6-file layout description needs updating to reflect `src/cli/`, `src/mcp/`, `src/index.ts`, `src/maintenance.ts`, and `src/embedded-skills.ts`.

### Phase 6: Idle Timeout PR Evaluation

Our unique contribution that upstream lacks. See assessment doc for the full comparison table.

**Key insight for next session**: The target is `src/mcp/server.ts` (not the old `src/mcp.ts`). The v2.0 MCP was rewritten as an SDK consumer — read it during Phase 4c and assess whether the idle timeout pattern still fits naturally. The implementation will be fresh code, not a rebase.

**Questions to answer**:
- Does v2.0's session model make idle timeout easier or harder to implement?
- Is there an existing issue on the upstream repo tracking session leak concerns?
- What's the right TTL for v2.0? (We used 5 minutes — may want to make it configurable)

---

## v2.0 Feature Testing Results

### `--explain` (score traces)

Works well. Returns full breakdown of how each result was scored:
- Per-backend scores (FTS BM25, vector cosine distance)
- Per-query RRF contributions with weights (original query gets 2x)
- Top-rank bonus (+0.05 for rank 1)
- Reranker score and final blended score
- Position-aware blending weights (ranks 1-3: 75% RRF / 25% reranker)

Useful for debugging search quality and understanding why a result ranked where it did.

### `--intent` (query disambiguation)

Significant impact on ambiguous queries. Testing with `"performance"`:
- **Without intent**: Expansion generated 5 queries (2 lex, 2 vec, 1 hyde) — wide net. Top result was Gemini CLI MCP documentation (tangentially related).
- **With intent `"search engine query latency"`**: Expansion generated only 1 hyde — narrow and targeted. Top result was distributed systems overview (directly relevant). Strong-signal bypass was skipped as designed.

The intent parameter steers all 5 pipeline stages as documented. The most notable effect is on expansion: the LLM generates fewer, more targeted queries when intent provides context.

### Cross-collection search

Works well. A query for "LaunchAgent deploy qmd mcp server" (no `-c` flag) returned results from 4 collections: qmd, projects-root-folder, qmd2, and kuato. The kuato result (its own LaunchAgent setup doc) was genuinely relevant cross-project context.

### Edges and Issues Found

| Issue | Severity | Detail |
|-------|----------|--------|
| **Query expansion repetition loop** | Medium | "MCP multi-client session management implementation" with intent generated **33 expansions** (16.4s!) — the model got stuck producing the same hyde/vec variations. Related to the v0.8.0 fix for "Qwen3 sampling params to prevent repetition loops" — may have regressed or the fix is incomplete for intent-augmented prompts |
| **HyDE `</think>` tag leak** | Low | The "session management" query's hyde expansion ended with `</think>` — model's internal reasoning format leaking into output. Cosmetic but suggests the expansion prompt could be tighter |
| **Duplicate results across mirror collections** | Low | The audit doc appeared from both `qmd` and `qmd2` collections. If `qmd2` is a mirror/backup, use `includeByDefault: false` or collection ignore patterns to exclude it from default searches |
| **Reranking latency** | Info | ~10s for 15-20 chunks on M3. This is the main query latency bottleneck. The v1.1.2 parallel reranking cap (4 contexts) may be tunable for M3's 11.8GB VRAM |

### sqlite-vec: First Time Working

The switch from Bun to Node resolved a long-standing issue. Under Bun, `bun:sqlite` doesn't support `loadExtension()`, so sqlite-vec never loaded. The store.ts code treats this as optional (line 635: "sqlite-vec is optional — vector search won't work but FTS is fine"), so qmd ran without errors — but:

- Vector search silently returned empty results
- Hybrid queries fell back to BM25-only (or hit the strong-signal shortcut)
- `qmd cleanup` crashed because `cleanupOrphanedVectors()` queries `vectors_vec` unconditionally (arguably a v2.0 bug — should guard like the rest of the codebase)

After switching to Node: `qmd cleanup` found and removed **78 orphaned embedding chunks** that had accumulated. Vector search is now active for the first time in this environment. Search quality for hybrid queries should be noticeably improved.

### SDK / Library Mode (`src/index.ts`)

Reviewed the 524-line SDK entry point. Key design:
- `QMDStore` interface: 4 search methods, 3 retrieval, collection/context CRUD, indexing, lifecycle
- Each store manages its own `LlamaCpp` instance (lazy-load, auto-unload after 5 min inactivity)
- `search()` accepts either `query` (auto-expand) or `queries` (pre-expanded) — the split enables inspect-and-filter workflows
- `expandQuery()` returns typed sub-queries you can curate before passing to `search({ queries })`
- MCP server is now a clean SDK consumer (zero internal store access)

This unblocks the documentor project's Document Adapter concept — `import { createStore } from '@tobilu/qmd'` for programmatic search without CLI shelling.

---

## Insights for Next Session

### v2.0 Architecture Changes to Understand

The biggest structural change is the **SDK layer** (`src/index.ts`). The old architecture had CLI and MCP both importing `store.ts` directly. Now:

```
src/index.ts (SDK — public API)
  └── src/store.ts (internal)
  └── src/maintenance.ts (new)

src/cli/qmd.ts → consumes SDK
src/mcp/server.ts → consumes SDK
```

This means MCP tools no longer call store functions directly — they go through the `QMDStore` interface. When reading `src/mcp/server.ts`, look for how tools invoke `search()`, `get()`, etc. via the SDK rather than raw store methods.

### New Files Worth Reading

| File | Why |
|------|-----|
| `src/index.ts` (524 lines) | The SDK entry point — defines `QMDStore` interface, `search()`, `createStore()` |
| `src/mcp/server.ts` | Rewritten MCP server — understand session model before LaunchAgent restart |
| `src/maintenance.ts` | New — may affect `qmd update` behavior |
| `src/embedded-skills.ts` | Powers `qmd skill install` |
| `test/sdk.test.ts` (1,286 lines) | Best documentation of SDK behavior — read the test names to understand the API surface |
| `bin/qmd` | **Already read.** Checks for `bun.lock`/`bun.lockb` → exec bun, else exec node. We deleted `bun.lock` to force the node path. Comment on line 22 literally describes our vec0 error |

### The `intent` Parameter

This is the most immediately useful new feature for our MCP callers. When Claude Code or Gemini CLI searches QMD, they can now pass `intent` to disambiguate. For example, searching "performance" with `intent: "database query optimization"` steers all five pipeline stages toward the right results. Test this during Phase 4c to see the quality improvement.

### Documentor Project Unblocked

The SDK mode (`import { createStore } from '@tobilu/qmd'`) means the documentor project's Document Adapter concept can now consume QMD programmatically. No more shelling out to the CLI. This is worth noting when you return to documentor work.

---

## Handoff Notes for Next Session (Phase 4c)

**Read this before starting.** These are easy-to-miss details about the current environment state.

### QMD MCP server is DOWN

The LaunchAgent was stopped (`launchctl bootout`) as part of Phase 4a. The MCP tools in Claude Code (the `qmd` plugin) will not work until either:
- The LaunchAgent is re-bootstrapped, OR
- You start the server manually: `qmd mcp --http --port 8181`

### Uncommitted changes in working tree

The following modifications are NOT committed to `dev`:
- `package.json` — zod pinned to exact `4.2.1` (was `^4.2.1`)
- `package-lock.json` — new file, generated by `npm install`
- `bun.lock` — **deleted** (this is intentional — its absence makes the bin wrapper pick node over bun)
- `docs/branch-assessment-2026-03-11-v2-upgrade-plan.md` — ongoing edits throughout session
- `docs/session-summary-2026-03-11-v2-upgrade.md` — this file

These should be committed before any branch operations.

### CLAUDE.local.md is hidden

Renamed to `CLAUDE.local.md.bak` so stale v1.1.0 instructions don't load into context. Phase 5 handles restoring and cleaning it up. Until then, only the upstream v2.0 `CLAUDE.md` loads — which means some instructions (like "Use Bun exclusively") are **wrong for our environment** since we switched to Node/npm.

### Runtime is Node, not Bun

- Binary: `/Users/rymalia/.nvm/versions/node/v24.12.0/bin/qmd`
- Old `~/.bun/bin/qmd` symlink was deleted
- `bin/qmd` wrapper picks node because there's no `bun.lock`
- **Do not run `bun install`** — it would recreate `bun.lock` and switch the runtime back to Bun, re-breaking sqlite-vec
- Use `npm install` / `npm run build` / `npm link` for all package operations

**Why this matters — the `bin/qmd` wrapper and `bun.lock`:**

The `bin/qmd` shell wrapper decides the runtime by checking if `bun.lock` or `bun.lockb` exists in the package directory. The upstream repo checks `bun.lock` into git (for the maintainer's Bun-based dev workflow), but the published npm package excludes it via the `"files"` whitelist in `package.json`. This means:

| Install method | Gets `bun.lock`? | Wrapper picks | sqlite-vec works? |
|---------------|-------------------|---------------|-------------------|
| `npm install -g @tobilu/qmd` (registry) | No | node | Yes |
| `bun install -g @tobilu/qmd` (registry) | No, but bun creates `bun.lockb` | bun | **No** — `bun:sqlite` lacks `loadExtension()` |
| Clone + `bun install` (source) | Yes (checked in) | bun | **No** — same reason |
| Clone + `npm install` (source) | Yes (checked in) | bun (wrong!) | **No** — ABI mismatch, npm compiled for node but wrapper runs bun |

We're in a modified state: cloned source, deleted `bun.lock`, installed via npm. This makes the wrapper pick node, and sqlite-vec works for the first time in this environment. Drafted as upstream issue — see `docs/issue-draft-bin-wrapper-source-builds.md`.

### LaunchAgent plist is likely compatible as-is

The plist PATH includes `/Users/rymalia/.nvm/versions/node/v24.12.0/bin`, which is exactly where the npm-linked `qmd` binary lives now. This was previously a concern (the plist was written for a different setup) but the runtime switch accidentally aligned the paths. Verify before re-bootstrapping, but it should work without plist edits.

### Filed upstream issues

- [#379](https://github.com/tobi/qmd/issues/379) — zod version range too loose (build failure)
- [#380](https://github.com/tobi/qmd/issues/380) — `qmd cleanup` crashes when sqlite-vec unavailable
- [#381](https://github.com/tobi/qmd/issues/381) — `bin/qmd` wrapper picks wrong runtime for source builds with npm
- All three open. See `docs/session-summary-2026-03-11-v2-assessment-and-plugin-audit.md` for full analysis of the issue chain and PR opportunities
