---
session_id: 80ce4d50-88e8-4d23-94bc-eddbd74fd128
date: 2026-08-13
time: "1:07 AM PDT – 2:09 AM PDT"
project: qmd
branch: dev
---

# Session Summary — Upstream Watchlist Cleared + dev Catch-up Merge

## Overview

The upstream maintainer (@tobi) ran a large merge spree on 2026-08-12/13, resolving
**every item on our tracked watchlist** (6 issues, 2 of our PRs, 1 community PR). This
session confirmed the resolutions, caught the local `dev` branch up to `upstream/main`
(`200bf98`) via a `--no-commit` merge with one conflict resolved, rebuilt and restarted
the source-mode MCP daemon, live-verified the three multi-get fixes, and refreshed the
now-stale memory and `CLAUDE.local.md` bug docs.

## Key Decisions Made

| Decision | Rationale |
|----------|-----------|
| **Resolve `marketplace.json` conflict to upstream's values** (`source: ./skills`, `version: 2.6.3`, adds `release` skill + `mcpServers` block) | Upstream's #831/#832 are the canonical relands of our own #794/#795 and are strictly more complete than the local `./skills/qmd` / `0.1.3` state; upstream also versions the plugin to match the software (better than incremental 0.1.x). |
| **Stash (not discard) the uncommitted local `marketplace.json` experiment** before merging | The working tree held a superseded hand-edit (`./skills` / `0.1.4`); stashing kept it recoverable through the merge. Dropped at end of session once confirmed redundant. |
| **Merge with `--no-commit`, hand the commit to the user** | Respects the standing "never run git commit" rule while still completing the mechanical integration and conflict resolution. |
| **Live-verify each multi-get fix rather than trust PR titles** | Per the "verify before claiming" preference; ran actual `multi-get` calls against the restarted daemon before rewriting docs to say the bugs are fixed. |
| **Flag, not fix, the unanchored-suffix edge from #759** | #868 unified path resolution but I did not independently reproduce the silent-wrong-match edge, so the doc marks that specific warning stale rather than asserting a fix. |

## Changes Made

| Change | Detail |
|--------|--------|
| **Catch-up merge** | Merged `upstream/main` (`200bf98`) into `dev` — 49 files, ~6.8k insertions (multi-get fixes, `mcp-pid`, `embed-lock`, `version.ts`, `release-context.sh`, many new tests). Commit `96a90d0`. |
| **Conflict resolution** | Rewrote `.claude-plugin/marketplace.json` to upstream's canonical block (valid JSON verified). |
| **Rebuild** | `npm run build` → exit 0; `dist/cli/qmd.js` (01:39) now newer than `src/`. |
| **Daemon restart** | `pkill` → LaunchAgent respawned at 01:40:34 (PIDs 65455/65459/65460), postdating the rebuild. |
| **Memory update** | Rewrote `memory/project_pr_watch_routine.md` to "watch complete / resolved"; updated `MEMORY.md` index line. |
| **Doc refresh** | `CLAUDE.local.md`: rewrote the multi-get "known gaps" section to reflect the fixes and updated the Version line to record the catch-up merge. Commit `374a279`. |
| **Cleanup** | Dropped the superseded `marketplace experiment` stash (`1c52038`). |

## Research / Verification Performed

- **Upstream status sweep** via `gh`: last 20 merged PRs, release/tag list, and the state
  of 6 tracked issues (#706, #759, #760, #789, #790, #796) and 8 PRs (#753, #794, #795,
  #831, #832, #846, #864, #868). Confirmed all issues CLOSED and our PRs relanded with
  `Original author: @rymalia` credit.
- **Merge compile check**: `npm run build` exit 0 validated the merged source before commit.
- **Live functional verification** against the restarted daemon:
  - `qmd multi-get "#dcbedc"` → resolves (was broken everywhere pre-merge) — #706/#753.
  - `qmd multi-get "qmd/docs/SYNTAX.md"` (collection-prefixed) → resolves in CLI (was "File not found") — #759/#868.
  - `qmd multi-get ... --format files` → `#1b5968,docs/SYNTAX.md,"..."`, docid as its own field — #760/#846.
  - MCP `status` tool → 4712 docs, vector index present, `needsEmbedding: 0`.
- **Post-merge parity**: `git diff upstream/main dev -- src/ bin/ test/ package.json` is empty — zero source divergence restored.

## Summary Statistics

- 6 issues + 8 PRs status-checked upstream; all 6 tracked issues confirmed resolved.
- 1 merge commit (49 files), 1 docs commit; both pushed to `origin/dev` (0/0 sync).
- 3 multi-get bug fixes verified live; 1 build (exit 0); 1 daemon restart verified.
- 3 memory/doc files edited (`project_pr_watch_routine.md`, `MEMORY.md`, `CLAUDE.local.md`).
- Local `main` fast-forwarded to `upstream/main` (was 77 behind).

## Discoveries / Handoff Notes

- **Both our PRs shipped under maintainer relands.** #794→#832 (plugin version bump) and
  #795→#831 (scope plugin to `./skills`); tobi closed ours as CONFLICTING (stale base) and
  relanded onto fresh `main` crediting `Original author: @rymalia`. Attribution preserved.
- **Still merged-but-unreleased.** Latest published release/tag is v2.5.3 (2026-05-29);
  package version stays 2.6.3. The plugin-version fix (#832) only benefits end users once
  an actual release bumps the plugin version.
- **nvm timebomb unchanged.** The LaunchAgent plist still hardcodes
  `/Users/rymalia/.nvm/versions/node/v24.18.0/bin`; a future Node upgrade removing that
  dir will break the daemon again.
- **`PowderAddicts` noise confirmed.** Every "1 comment" on the tracked threads is that
  helpdesk auto-responder, never real review — the monitor's exclusion of it held up.

## Current State

- `dev` = `374a279`, clean tree, pushed to `origin/dev` (in sync). Zero source divergence
  from `upstream/main`. Local `main` even with `upstream/main`.
- MCP daemon running merged source-mode code (started 01:40:34, PIDs 65455/65459/65460).
- The `qmd-watch-upstream` cloud routine (`trig_01WbeXf222N9L9RADtsEBW4p`) was disabled by
  the user this session — nothing left on its watchlist.
- Remaining `stash@{0}` is a pre-existing unrelated doc-sprint WIP, not from this session.

## Issues & PRs (all upstream `tobi/qmd`, resolved 2026-08-12/13)

- #706 → PR #753 (@dpersek) — https://github.com/tobi/qmd/pull/753
- #759 → PR #868 (tobi, unify multi-get comma-list path resolution) — https://github.com/tobi/qmd/pull/868
- #760 → PR #846 (tobi, docid as own `--format files` field) — https://github.com/tobi/qmd/pull/846
- #789 → PR #832 (reland of our #794) — https://github.com/tobi/qmd/pull/832
- #790 → PR #831 (reland of our #795) — https://github.com/tobi/qmd/pull/831
- #796 → PR #864 (add release-context.sh) — https://github.com/tobi/qmd/pull/864
- Our originals, closed CONFLICTING: #794 https://github.com/tobi/qmd/pull/794 · #795 https://github.com/tobi/qmd/pull/795

## Unfinished Work

None. All planned work completed and committed. Optional future consideration only: a new
upstream release would finally publish the merged-but-unreleased v2.6.3 line (out of our
control), and the nvm-path timebomb in the LaunchAgent plist remains a latent risk to
address whenever Node is next upgraded.
