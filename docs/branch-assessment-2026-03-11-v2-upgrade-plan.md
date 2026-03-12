# QMD Branch Assessment & v2.0 Upgrade Plan

**Date:** 2026-03-11
**Current version:** v1.1.0 (local) | v2.0.1 (upstream)
**Current branch:** `dev`
**Author:** rymalia + Claude

---

## Purpose

This document captures the state of all active branches in our QMD fork before pulling upstream's v2.0 release. It serves as a reference for what work existed, what's been superseded, and the plan for safely updating without losing anything.

---

## Branch Overview

| Branch | Status | Commits ahead of `main` | Purpose |
|--------|--------|-------------------------|---------|
| `main` | Clean mirror of upstream | 0 | Tracks `upstream/main` (tobi/qmd) |
| `dev` | Active working branch | 8 | Aggregates feature work + local docs |
| `feature/mcp-multi-client` | PR candidate (now superseded) | 3 | Per-client MCP HTTP session management |
| `feat/finetune-mlx-transition` | Parked | (not assessed) | MLX finetuning migration — out of scope for this plan |

---

## Branch: `main`

Our `main` is a clean fast-forward of `upstream/main`. It has **zero commits** that aren't in upstream. This means updating main is trivial: `git merge upstream/main` will fast-forward cleanly.

**Last synced to:** v1.1.0 (commit `0e0feb6`, upstream's `release: v1.1.0`)

---

## Branch: `dev` (8 commits ahead of `main`)

### Commit History

| Hash | Date | Description | Category |
|------|------|-------------|----------|
| `ef847a4` | 2026-02-24 | docs: MCP HTTP migration guide and Node package manager reference | Docs |
| `756c89e` | 2026-02-25 | **feat(mcp): support multiple concurrent HTTP clients** | Feature |
| `3430733` | 2026-02-25 | docs: multi-agent collaboration on mcp multi-client | Docs |
| `72134a4` | 2026-02-25 | docs: node and package managers reference | Docs |
| `652fecc` | 2026-02-25 | fix(build): pin zod to 4.2.1 to resolve tsc type errors in mcp.ts | Fix |
| `7bc12e2` | 2026-02-26 | docs: correct MCP session model and add server lifecycle guide | Docs |
| `fa3cf12` | 2026-03-01 | Merge branch 'feature/mcp-multi-client' into dev | Merge |
| `29209f7` | 2026-03-01 | docs: document multi-client deployment pipeline and fix stale paths | Docs |

### Files Changed (vs `main`)

| File | Lines Changed | Notes |
|------|---------------|-------|
| `CLAUDE.md` | +82 | Project-specific instructions (local only, not PR material) |
| `GEMINI.md` | +73 | Gemini CLI agent instructions (local only) |
| `README.md` | +138 | Extended documentation |
| `bun.lock` | net -34 | Dependency changes from zod pin (will be fully regenerated on v2.0 — not a cherry-pick candidate) |
| `package.json` | +2/-1 | Zod pinned to 4.2.1 |
| `src/mcp.ts` | +87 | **Core feature: multi-client session management** |
| `finetune/reward.py` | +22/-47 | Simplification: removed `INTERIOR_FILLER_WORDS`, simplified `extract_named_entities` compound chaining — net deletion |
| 7 session summary docs | +1,237 | Local development documentation |

### What's Worth Keeping

| Content | Keep? | Reason |
|---------|-------|--------|
| `src/mcp.ts` multi-client changes | **No** | Superseded by upstream PR #286 |
| `package.json` zod pin | **No** | v2.0 has new dependency tree |
| `CLAUDE.md` additions | **Yes** | Local project instructions, not for upstream |
| `GEMINI.md` | **Yes** | Local agent instructions |
| `README.md` extensions | **No** | Upstream README rewritten for v2.0 SDK docs; our additions would conflict and are stale |
| Session summary docs | **Yes** | Our development history |
| `docs/node-101-package-managers.md` | **Yes** | Reference doc |
| `finetune/reward.py` changes | **Drop** | Upstream commit `4511b9b` tightens entity detection and adds filler penalty — opposite direction from our simplifications. Upstream's reward.py has moved well past both versions |

---

## Branch: `feature/mcp-multi-client` (3 commits ahead of `main`)

### Intent

This branch was created to isolate the multi-client MCP feature for a clean PR to upstream. It's a subset of `dev`, containing only the feature commit and its direct documentation.

### Commits

| Hash | Date | Description |
|------|------|-------------|
| `ef847a4` | 2026-02-24 | docs: MCP HTTP migration guide and Node package manager reference |
| `756c89e` | 2026-02-25 | feat(mcp): support multiple concurrent HTTP clients |
| `4ca0d5a` | 2026-03-01 | docs: document multi-client deployment pipeline and fix stale paths |

### The Feature: What We Built

The problem: QMD's HTTP MCP server created a single `McpServer` + `Transport` at startup. Once one client initialized, all others got "Server already initialized" — making HTTP mode single-client only.

Our solution (commit `756c89e`):
- **Per-client session map**: `Map<string, Session>` where each session holds its own `McpServer` + `Transport` pair
- **Session routing**: incoming requests routed by `mcp-session-id` header
- **Idle timeout cleanup**: 5-minute TTL with a 60-second cleanup interval (`setInterval`)
- **`lastAccess` tracking**: each request updates the session's timestamp
- **Graceful error handling**: stale/expired session IDs return structured JSON-RPC errors

### Status: Superseded by Upstream

**Upstream PR #286** (`joelev/fix/multi-session-http`, merged 2026-03-07) implements the same feature with the same architecture. Key differences:

| Aspect | Ours (`756c89e`) | Upstream (`383a2e5`) |
|--------|-------------------|----------------------|
| Session storage | `Map<string, Session>` (server + transport + lastAccess) | `Map<string, Transport>` (transport only) |
| Session creation | Manual UUID + immediate map insertion | `onsessioninitialized` callback (cleaner) |
| Idle cleanup | 5-min timeout via `setInterval` | None — relies on `onclose` only |
| Initialize detection | Checks `body.method === "initialize"` | Uses SDK's `isInitializeRequest()` helper |
| Stale session response | `"Session expired"` (code -32600) | `"Session not found"` (code -32001) |
| Missing session-id | Creates new session | Returns 400 error |

### What We Have That Upstream Doesn't

**Idle session timeout** — our implementation proactively cleans up sessions that go silent (client crashes, network drops, etc.). Upstream relies solely on the client sending a DELETE request or the transport's `onclose` event firing. If neither happens, upstream leaks the session indefinitely.

This is a genuine improvement worth contributing as a **small follow-up PR** to upstream.

---

## Upstream Changes: v1.1.0 to v2.0.1 (61 commits)

### Version-by-Version (from CHANGELOG.md)

| Version | Date | Key Changes |
|---------|------|-------------|
| **v1.1.1** | 2026-03-06 | Reranker truncation fix (2048-token context), Nix build deps |
| **v1.1.2** | 2026-03-07 | 13 community PRs: GPU autodetect via node-llama-cpp, `--explain` score traces, collection ignore patterns, multilingual embeddings (`QMD_EMBED_MODEL`), configurable expansion context, `candidateLimit` exposed, **MCP multi-session (#286)**, rerank perf improvements |
| **v1.1.5** | 2026-03-07 | `intent` parameter for query disambiguation — steers all 5 pipeline stages (expansion, strong-signal bypass, chunk selection, reranking, snippet extraction). Design by @vyalamar |
| **v1.1.6** | 2026-03-09 | **SDK/library mode**: `createStore()` for programmatic access, package exports for bundlers |
| **v2.0.0** | 2026-03-10 | **Major refactor**: source split into `src/cli/` + `src/mcp/`, stable `QMDStore` SDK interface, unified `search()` API (replaces query/search/structuredSearch split), MCP rewritten as clean SDK consumer, `better-sqlite3` ^12.4.5 for Node 25, runtime-aware bin wrapper |
| **v2.0.1** | 2026-03-10 | `qmd skill install` command, Qwen3-Embedding filename case fix, symlinked launcher path fix |

### Our Multi-Client Feature in Upstream's Timeline

The multi-client problem had three touchpoints in the upstream history:

1. **v1.1.2** (2026-03-07): PR #286 by @joelev merged — same fix we built, implemented independently. Credited in changelog as "MCP multi-session: HTTP transport now supports multiple concurrent client sessions"
2. **v2.0.0** (2026-03-10): Entire MCP server **rewritten from scratch** as an SDK consumer in `src/mcp/`. The v1.1.2 multi-session code was absorbed into the new architecture
3. **Current state**: Multi-session is baked into v2.0's MCP. Any idle-timeout PR would target the new `src/mcp/` code, not the old `src/mcp.ts`

### Notable Community Contributions (v1.1.0 → v2.0.1)

| PR | Author | Feature |
|----|--------|---------|
| #286 | @joelev | Multi-session HTTP (same as our feature) |
| #242 | @vyalamar | `--explain` score traces for hybrid search |
| #180 | @vyalamar | `intent` parameter design |
| #273 | @daocoding | Configurable embedding model |
| #304 | @sebkouba | Collection ignore patterns |
| #255 | @pandysp | Expose candidate limit |
| #313 | @0xble | Configurable expansion context size |
| #247 | — | Quoted phrases, negation, entity preservation in finetuning |
| #355 | @nibzard | Skill installer |

### Breaking Changes in v2.0.0

1. **Source directory restructure**: `src/mcp.ts` → `src/mcp/` subdirectory; `src/qmd.ts` → `src/cli/`
2. **SDK API**: New `QMDStore` interface, unified `search()` replaces the old query/search/structuredSearch split
3. **MCP rewritten**: Now consumes the SDK rather than importing store directly — zero internal store access
4. **Dependency changes**: better-sqlite3 bumped to ^12.4.5, runtime-aware bin wrapper added
5. **Removed MCP tools** (already gone as of our v1.1.0): `search`, `vector_search`, `deep_search` were removed in v1.1.0 — not a new break from this upgrade, but worth noting since older skill docs may still reference them. Only `query`, `get`, `multi_get`, `status` remain

### Impact on Our Local Configuration

These upstream changes affect files and settings we maintain locally:

| Our Config | Issue | Action Needed |
|------------|-------|---------------|
| Skill files (`skills/qmd/SKILL.md`, `skills/qmd/references/mcp-setup.md`) | Reference stale tool name `structured_search` (renamed to `query` in v1.1.0), outdated install paths, and incorrect version numbers (identified in Mar 1 audit) | Update skill files after upgrade |
| `src/mcp.ts` modifications | File no longer exists in v2.0 (now `src/mcp/`) | Drop — feature is upstream |
| `package.json` zod pin | v2.0 has entirely new dependency tree | Drop — reassess after install |
| LaunchAgent plist | Binary path unchanged (`~/.bun/bin/qmd`) but rebuild required | Restart after `bun link` |

This means our `src/mcp.ts` diff will **not merge cleanly** against v2.0 — but since the feature is already upstream, that's fine.

---

## What v2.0 Gives Us

Beyond catching up with bug fixes and community contributions, this upgrade unlocks features that directly benefit our workflow and the broader project ecosystem:

| Feature | Version | Why It Matters |
|---------|---------|----------------|
| **SDK/library mode** (`import { createStore } from '@tobilu/qmd'`) | v1.1.6, stabilized v2.0.0 | The documentor project's Document Adapter concept can now use QMD programmatically — no CLI shelling needed. This is the single most important upstream addition for our ecosystem |
| **`intent` parameter** | v1.1.5 | Our MCP callers (Claude Code, Gemini CLI) can disambiguate queries by passing intent context, directly improving search quality for ambiguous terms |
| **`--explain` score traces** | v1.1.2 | Transparent debugging of search quality for our collections — see backend scores, RRF contributions, reranker scores, and final blended scores |
| **Collection ignore patterns** | v1.1.2 | Can exclude noise files (sessions, tmp) from indexing via `ignore: ["Sessions/**", "*.tmp"]` in collection config |
| **Configurable embedding model** (`QMD_EMBED_MODEL`) | v1.1.2 | Swap in multilingual models like Qwen3-Embedding for non-English content |
| **`candidateLimit` exposed** | v1.1.2 | Tune reranker throughput via CLI flag `-C` or MCP parameter |
| **`qmd skill install`** | v2.0.1 | One-command skill setup into `~/.claude/commands/` |
| **Unified `search()` API** | v2.0.0 | Replaces the old query/search/structuredSearch split — simpler programmatic interface |
| **Runtime-aware bin wrapper** | v2.0.0 | Detects bun vs node to avoid ABI mismatches — addresses the exact issue we documented in our Node 101 reference |

---

## The Plan

### Phase 0: Safety Tags (Preserve Everything)

```bash
git tag archive/dev-pre-v2-merge dev
git tag archive/feature-mcp-multi-client feature/mcp-multi-client
```

These lightweight tags permanently bookmark the current commit at the tip of each branch. Even after rebasing or deleting branches, the full history is recoverable:

```bash
git checkout archive/dev-pre-v2-merge       # view old state
git checkout -b recovery archive/dev-pre-v2-merge  # restore as branch
git tag -d archive/dev-pre-v2-merge         # delete when no longer needed
```

### Phase 1: Update `main`

```bash
git checkout main
git merge upstream/main   # fast-forward, no conflicts expected
```

This brings main from v1.1.0 to v2.0.1 (61 commits).

### Phase 2: Assess `dev` Against Updated `main`

After main is current, evaluate each dev commit:

- **Drop**: `756c89e` (multi-client) — upstream has it
- **Drop**: `652fecc` (zod pin) — v2.0 has new deps
- **Keep**: CLAUDE.md, GEMINI.md additions — local project config
- **Keep**: Session summary docs — our development history
- **Drop**: `finetune/reward.py` — upstream's `4511b9b` went the opposite direction (tightened entity detection, added filler penalty vs our simplifications)
- **Drop**: `README.md` extensions — upstream README was significantly rewritten for v2.0 SDK docs; our additions would conflict and are mostly stale

### Phase 3: Rebuild `dev` on Updated `main`

The keepers from `dev` are almost entirely additive files that don't exist upstream — local config (`CLAUDE.md`, `GEMINI.md`), session summaries, and reference docs. None of them touch upstream source code. This means cherry-picking individual commits adds complexity for no benefit — the commits mixed kept and dropped files (e.g. `ef847a4` includes both docs and the MCP migration guide that references stale paths).

**Recommended approach**: copy the files directly from the old branch and make a single clean commit.

```bash
git checkout main
git branch -m dev dev-old           # rename current dev
git checkout -b dev                 # new dev from updated main

# Copy keepers from old dev
git checkout dev-old -- GEMINI.md
# NOTE: CLAUDE.md handled separately in Phase 3b
git checkout dev-old -- docs/session-summary-*.md
git checkout dev-old -- docs/node-101-package-managers.md
git checkout dev-old -- docs/audit-2026-03-01-session-docs-accuracy.md
git checkout dev-old -- docs/pr-review-mcp-multi-client.md
git checkout dev-old -- docs/branch-assessment-2026-03-11-v2-upgrade-plan.md

# Stage and commit as a single restore
git add -A
git commit -m "docs: restore local project docs from pre-v2 dev branch"
```

**Why not cherry-pick**: The 8 dev commits interleave kept content (docs, CLAUDE.md) with dropped content (mcp.ts changes, zod pin, reward.py). Cherry-picking would require `--no-commit` plus manual unstaging of dropped files for almost every commit — more error-prone than a clean file copy.

**Files intentionally NOT copied**: `src/mcp.ts` (superseded), `package.json` zod pin (new dep tree), `bun.lock` (regenerated), `finetune/reward.py` (upstream diverged), `README.md` (upstream rewritten).

### Phase 4: Clean Install of v2.0

**Do this before touching CLAUDE files or LaunchAgent config.** The goal is a vanilla v2.0 running in its default configuration, so we can evaluate what's changed before layering customizations back on.

#### 4a: Stop LaunchAgent and clear local customizations

```bash
# Stop the LaunchAgent so nothing holds the old binary or port
launchctl bootout gui/$(id -u)/com.qmd.mcp

# Verify port is free
lsof -i :8181

# Hide CLAUDE.local.md from context (postpone cleanup to Phase 5)
mv CLAUDE.local.md CLAUDE.local.md.bak
```

#### 4b: Build and install

```bash
# Pre-flight: v2.0 bumped better-sqlite3 for Node 25 support.
# If the runtime-aware bin wrapper resolves to node, it needs >= 25.
node --version   # verify >= 25, or confirm bun is the active runtime

bun install
bun run build
bun link

# Verify
qmd status
```

**Note:** The `bun.lock` will be entirely regenerated from v2.0's dependency tree — this is not a cherry-pick decision, it's a full replacement.

#### 4c: Assess MCP and LaunchAgent

After install, examine the v2.0 MCP server before re-enabling the LaunchAgent:
- Read the new `src/mcp/` source to understand v2.0's session management and daemon behavior
- Check if v2.0 has its own daemon management story that supersedes our LaunchAgent plist
- Test the MCP server manually (`qmd mcp --http`) before automating via LaunchAgent
- Only re-bootstrap the LaunchAgent once you've confirmed the plist paths and env vars are still correct for v2.0

### Phase 5: Clean Up CLAUDE.md / CLAUDE.local.md Duplication

Both `CLAUDE.md` (checked-in, shared) and `CLAUDE.local.md` (private, gitignored) are loaded into context every conversation. They currently have significant overlap — CLI reference, architecture, development commands, and restrictions appear in both, costing tokens for no benefit.

Additionally, v2.0 restructured the source (`src/cli/` + `src/mcp/`, `QMDStore` SDK interface), so the architecture sections in both files are now stale.

**Strategy**: Start fresh from upstream's v2.0 `CLAUDE.md`, then assess what to re-add. This is done *after* Phase 4 so we know what v2.0 actually looks like before deciding what local content to keep.

```bash
# On the new dev branch:
mv CLAUDE.md CLAUDE-temp.md              # preserve our version for reference
git checkout main -- CLAUDE.md           # restore upstream's v2.0 version
mv CLAUDE.local.md.bak CLAUDE.local.md   # restore for side-by-side review
```

Then review `CLAUDE-temp.md` and `CLAUDE.local.md` side by side:
- **Upstream `CLAUDE.md`**: accept as-is — it reflects v2.0's structure and conventions. Only add back local operational content (e.g. MCP LaunchAgent deployment) if it's still accurate after the upgrade
- **`CLAUDE.local.md`**: strip everything that duplicates `CLAUDE.md`. Keep only genuinely personal content: agent preferences, workflow-specific restrictions, and any operational knowledge not in upstream's version
- **Delete `CLAUDE-temp.md`** once the review is done

**Goal**: zero duplication between the two files. `CLAUDE.md` = project conventions (shared). `CLAUDE.local.md` = personal setup and preferences (private).

### Phase 6: Evaluate Idle Timeout PR

Our idle session cleanup (5-min TTL, 60s interval) addresses a real gap in upstream's implementation. This evaluation benefits from Phase 4c — by then you'll have read the v2.0 MCP source and understand the new session model. Consider submitting as a focused PR against `upstream/main`:

- **Important**: the target file is now `src/mcp/` (v2.0 structure), not `src/mcp.ts`. The entire MCP server was rewritten as an SDK consumer in v2.0.0, so this needs to be implemented fresh against the new code — not rebased from our old commit
- Read the v2.0 MCP source first to understand the new session management pattern
- Add a test case (client connects, idles past TTL, verify cleanup)
- Keep the PR minimal — just the timeout, no other changes
- Reference the original problem: if a client crashes without sending DELETE or triggering `onclose`, the session leaks indefinitely

---

## Working Tree State (Not Committed)

These must be handled before any branch operations (checkout, cherry-pick, rebase) to avoid losing work or triggering conflicts.

### Untracked files

```
docs/audit-2026-03-01-session-docs-accuracy.md
docs/branch-assessment-2026-03-11-v2-upgrade-plan.md  ← this document
docs/feature-plan-finetune-mlx-transition.md
docs/gemini-cli-mcp-servers-documentation.md
docs/llm-architecture-deep-dive.md
docs/overview-llm-architecture.md
docs/pr-review-mcp-multi-client.md
finetune/GEMINI.md
finetune/configs/mlx_sft.yaml
finetune/data/mlx/
```

### Modified tracked files

```
CHANGELOG.md  (downloaded upstream's current changelog — will conflict on branch switch)
```

**Before proceeding with Phase 1**, either commit these to `dev`, stash them (`git stash --include-untracked`), or move them to a safe location outside the repo.

---

## Session History (For Context)

The work on these branches spanned several sessions:

| Date | Session | Focus |
|------|---------|-------|
| 2026-02-21 | M3 MLX transition | Finetuning pipeline exploration |
| 2026-02-24 | MCP HTTP migration | Migrated MCP from stdio to HTTP, discovered multi-client bug |
| 2026-02-24 | Gemini agent analysis | Multi-agent collaboration research |
| 2026-02-24 | Gemini MCP multi-client | Cross-agent investigation of the multi-client problem |
| 2026-02-25 | Observer MCP multi-client | Implemented the multi-client feature (commit `756c89e`) |
| 2026-02-25 | Zod fix and docs | Pinned zod to fix tsc errors, documentation cleanup |
| 2026-03-01 | LaunchAgent v1.1.0 deploy | Deployed to LaunchAgent, wrote deployment docs, PR review |
| 2026-03-01 | Documentation audit | Audited session summaries for accuracy |
