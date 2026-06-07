---
date: 2026-06-06
time: "7:37 PM PDT – 7:58 PM PDT"
project: qmd
branch: dev
---

# Session Summary: Install Method Reminder & Version Verification

## Overview

Documented the local `qmd` install method as a fast-recall topline note, re-verified the running version (v2.5.3) and the Bun runtime claim against the live binary, realigned a stale `dist/` build, and codified a required version-checking procedure into `CLAUDE.local.md`.

## Key Decisions Made

- **Topline reminder as a pointer, not a duplicate.** Added a TL;DR callout at the top of `CLAUDE.local.md` that answers "which install method?" at a glance and links down to the canonical `Current Install State` section — avoiding a second copy of the version number that would drift.
- **Honest verification over claimed verification.** Refused to bump the Runtime note's "as of v2.5.2" version without actually re-running the checks. Re-verified under Bun and recorded real evidence.
- **Did not run `qmd embed`.** Respected the CLAUDE.md ban on auto-running `embed`/`update`/`collection add`. Validated embedding output via `doctor`'s freshness + vector-sample checks and a live `query` instead of a destructive re-embed.
- **Build is safe to run; embed is not.** Distinguished `npm run build` (touches `dist/`, safe, also serves as a typecheck) from the banned heavy ops, and ran it only after explicit user go-ahead.
- **Procedure lives in `CLAUDE.local.md`, not private memory.** Chose the project's authoritative local-env doc (loaded every session) so the version-check procedure travels with the project rather than depending on agent recall.

## Changes Made

| Change | Detail |
|--------|--------|
| **Topline install reminder** | Added a TL;DR callout under the `CLAUDE.local.md` header: runs from local source via `npm link`, not the published `@tobilu/qmd` npm package; links to `Current Install State` |
| **Version bump** | `Current Install State` → `v2.5.3 (source build on dev branch, commit a9a8844, last synced 2026-06-06)` |
| **Runtime note re-verified** | Updated "Status as of v2.5.2 (verified 2026-05-26)" → "v2.5.3 (re-verified 2026-06-06)" with concrete evidence; noted `embed` was not re-run and why that's still sufficient |
| **Version-check procedure** | Added new `### Determining the current version (REQUIRED procedure)` subsection mandating a three-way cross-check (`qmd --version` / `package.json` / git HEAD) plus a `dist/` freshness check |
| **dist/ rebuild** | Ran `npm run build`; `dist/cli/qmd.js` realigned (19:57) ahead of `src/cli/qmd.ts` (19:27); pulled 2.5.3 code typechecks clean |

## Testing / Research Performed

- **Install-method confirmation:** `which qmd`, `npm -g list`, `qmd --version` — all three agree on local-symlink install at v2.5.3 (a9a8844).
- **`qmd doctor` (Node path):** `Runtime: better-sqlite3`, SQLite 3.51.3, sqlite-vec v0.1.9, GPU Metal probe (Apple M3, 11.8 GB VRAM), 3,155 docs on embedding fingerprint c37385, vector sample reproduces.
- **`qmd doctor` under Bun:** Forced `bun src/cli/qmd.ts doctor` → `Runtime: bun:sqlite`, SQLite 3.53.2, sqlite-vec v0.1.9, embeddings and vector sample valid — confirming the Bun path works on 2.5.3.
- **End-to-end pipeline under Bun:** `vsearch` ran (generated 4 vector queries, executed search); `query` returned a live `Score: 56%`, proving qwen3-reranker + embeddinggemma (node-llama-cpp) work end-to-end.
- **Build-freshness diagnosis:** `stat` on `dist/cli/qmd.js` vs `src/cli/qmd.ts` exposed a stale `dist/` (May 26) while the running version was 2.5.3 — the smoking gun proving the launcher runs `src/` via tsx.
- **Launcher inspection:** Read `bin/qmd` source-selection logic (lines 20–135) confirming tsx → `src/cli/qmd.ts` preference in a git checkout.
- **Post-build verification:** Re-`stat` confirmed `dist/` rebuilt ahead of `src/`; `qmd --version` still clean at 2.5.3.

## Summary Statistics

- Files modified: 1 (`CLAUDE.local.md` — 4 edits across header, install state, runtime note, and a new procedure subsection)
- Build artifacts: `dist/` recompiled to 2.5.3 (typecheck clean)
- Verification commands run: ~10 (install checks, dual-runtime `doctor`, vsearch, query, stat, launcher inspection)
- Code/source changes: 0 (documentation + build only)

## Discoveries / Handoff Notes

- **A bare `git pull` updates the live `qmd` version with no build/install/link step.** Because `npm link` symlinks the whole repo directory and the `bin/qmd` launcher runs `src/cli/qmd.ts` via tsx (not `dist/`) in a git checkout, pulling new `package.json` + `src/` goes live instantly. The `qmd --version` string reads `package.json` plus live git HEAD.
- **`dist/` can silently lag the running version.** It doesn't affect execution today (launcher prefers `src/`), but `dist/` is the fallback if tsx is ever unavailable — a stale `dist/` would mean a silent downgrade to old code. Always cross-check `dist/` freshness when asked about the current version.
- **The interactive `qmd` wrapper currently resolves to the Node path** (`better-sqlite3`), even though Bun 1.3.10 and `bun.lock` are both present. The Bun path (`bun:sqlite`) only engaged when invoked explicitly via `bun src/cli/qmd.ts`. Worth understanding the launcher's routing if Bun-vs-Node behavior ever matters.
- **Count discrepancy (unresolved, low priority):** `qmd doctor` reports 55 collections / 3,155 docs on fingerprint; the MCP server instructions list ~62 collections / 3,354 docs. Likely inactive/soft-deleted docs plus the daemon running an older snapshot than the interactive Bun invocation. Not reconciled this session.

## Current State

- **Running version:** `qmd 2.5.3 (a9a8844)` on branch `dev`.
- **Install:** local source at `/Users/rymalia/projects/qmd` via `npm link`; binary at `/Users/rymalia/.nvm/versions/node/v24.12.0/bin/qmd`.
- **`dist/`:** freshly rebuilt and aligned with `src/` (both 2.5.3).
- **MCP daemon:** LaunchAgent `com.qmd.mcp` (port 8181) — not touched this session; still serving its existing snapshot.
- **Uncommitted:** `CLAUDE.local.md` edits and the regenerated `dist/` are unstaged. Per project policy, the user performs commits.

## Unfinished Work

- **Optional:** Reconcile the collection/doc count discrepancy between `qmd doctor` (55/3,155) and the MCP server (62/3,354) if it matters — likely a daemon-snapshot age difference.
- **Optional:** Restart the MCP LaunchAgent (`launchctl unload`/`load`) if you want the daemon picking up the freshly-built 2.5.3 `dist/` — though source-mode means it already runs current `src/` on restart.
- **Commit:** The user may want to commit the `CLAUDE.local.md` documentation updates. (`dist/` is typically build-output and may be gitignored — verify before staging.)
