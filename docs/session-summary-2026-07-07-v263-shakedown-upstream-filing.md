---
session_id: 3cd37fe7-69ad-4586-86d4-93b8b9eeb767
date: 2026-07-07
time: "2026-07-06 11:58 PM PDT – 2026-07-07 2:20 AM PDT"
resumed: "12:14 AM PDT, 1:11 AM PDT"
project: qmd
branch: main
---

# Session Summary: Daemon Repair, v2.6.3 Upgrade Shakedown & Upstream Filing

## Overview

Diagnosed and fixed a dead QMD MCP daemon (stale nvm path in the LaunchAgent plist), shepherded the checkout from `dev`/v2.5.3 to `main`/v2.6.3, ran a full CLI + MCP shakedown of the new version, re-validated every version-specific claim in CLAUDE.local.md (~14 edits), and — after a second-review cycle with the dev analyst — filed two novel upstream issues and posted two supporting comments on the existing docid issue/PR.

## How We Got Here (session arc)

1. **"Is my qmd mcp running?"** → No. LaunchAgent loaded but process dead (exit status 78, port 8181 refusing).
2. **Root cause**: Node upgraded v24.12.0 → v24.18.0 around Jun 30; the old nvm directory was deleted and the plist hardcodes both the binary path and PATH. Fixed both, reloaded, verified with a real MCP `initialize`.
3. **Branch migration**: user moved `main` forward 47 commits (v2.5.2 → v2.6.3) and used `git restore --source=origin/dev CLAUDE.local.md` to keep the (gitignored-on-main) local instructions file on disk.
4. **The version lie**: after the user's rebuild, the daemon *reported* 2.6.3 but was still the 00:01 process running pre-switch code from memory — `serverInfo.version` reads `package.json` live at request time. Detected via process start time (`ps -o lstart`), restarted cleanly (00:45:31), re-verified.
5. **Shakedown + revalidation** (user request, with `@docs/SYNTAX.md` folded in): exercised every search/retrieval surface and audited CLAUDE.local.md claim-by-claim against v2.6.3 source.
6. **Second review**: analyst validated the work, found 3 gaps (all accepted), and expanded the multi-get finding into a 5-row repro matrix with CLI/MCP divergence.
7. **Upstream recon + filing**: analyst's repo scan found our docid issue was a dup (#706, fix in flight as PR #753) but the path-matching issue was novel. Filed #759 and #760; commented on #706 and #753.

## Key Decisions Made

- **Split the upstream report into two issues** — docid support (missing feature) vs. path-matching semantics (divergence + silent wrong-document hazard) have different fixes, seams, and severities. Vindicated when the docid half turned out to be already-filed (#706).
- **Contribute to #706/#753 rather than duplicate** — posted the generalized matrix as a comment on #706 and a scope-confirming plain comment (not a formal review, per user guidance) on #753.
- **Kept the daemon's LaunchAgent on Node+tsx** — no runtime changes; v2.6.3's lockfile-driven launcher pins everything to Node anyway (local untracked `package-lock.json` beats tracked `bun.lock`).
- **`qmd://` URIs documented as the universal safe form** for multi-get comma-lists — the only pattern shape that works in every branch of both implementations.

## Changes Made

| Change | Detail |
|--------|--------|
| **LaunchAgent plist repaired** | `~/Library/LaunchAgents/com.qmd.mcp.plist`: both v24.12.0 paths → v24.18.0; unload/load; daemon healthy on port 8181 |
| **Daemon restarted onto v2.6.3 code** | Caught the stale-process-reporting-new-version trap; clean `launchctl` cycle; verified via process start time + MCP initialize |
| **CLAUDE.local.md — install state** | Version line → v2.6.3/main/e428df7; runtime lines → v24.18.0 with upgrade history |
| **CLAUDE.local.md — version procedure** | Added "the running daemon's reported version can lie" check (process start time is the truth) |
| **CLAUDE.local.md — pkill fix** | Documented pattern `qmd.js mcp --http` matched nothing in source mode; fixed to `cli/qmd.[tj]s mcp --http` (both places) |
| **CLAUDE.local.md — runtime section** | Rewritten for v2.6.3: launcher now cross-platform Node script, lockfile-driven runner selection, Bun-interactive claim obsoleted, `GGML_METAL_NO_RESIDENCY` launcher behavior, `npm test` = test-all.mjs (typecheck+node+bun+smoke) |
| **CLAUDE.local.md — architecture** | `src/embedded-skills.ts` removed (skills live in `skills/` dir); `store_collections` table added; pipeline constants stamped re-verified |
| **CLAUDE.local.md — known gaps section** | Full multi-get comma-list matrix (CLI vs MCP divergence, unanchored LIKE, arbitrary LIMIT 1), qmd:// workaround, `--format files` parseability defect, upstream issue numbers |
| **CLAUDE.local.md — plugin layer** | Marketplace revert to `github:tobi/qmd` now unblocked (#718/#719 merged) |
| **Memory: reference_launchagent.md** | v24.18.0 path + warning that plist pins the nvm version (exit 78 on Node upgrades) |
| **Memory: project_pr_watch_routine.md** | #718/#719 MERGED, #716 still open; routine ready to disable/retarget |
| **Memory: feedback_verify_before_claiming.md** (new) | Two claim-hygiene rules from the review (see Discoveries) |
| **Memory: MEMORY.md** | Index updated for both memory changes |

## Issues & PRs (filed/commented this session)

- **Filed** [tobi/qmd#759](https://github.com/tobi/qmd/issues/759) — multi-get comma-lists: CLI/MCP divergent path resolution; unanchored suffix match can silently fetch the wrong document; arbitrary `LIMIT 1` collection resolution. With 5-row repro matrix.
- **Filed** [tobi/qmd#760](https://github.com/tobi/qmd/issues/760) — multi-get `--format files` prepends docid inside the first CSV field, breaking naive comma-splitting.
- **Commented on** [#706](https://github.com/tobi/qmd/issues/706#issuecomment-4902197390) — confirmed repro on 2.6.3 `e428df7` + contributed the generalized matrix (our docid issue draft was retired as a dup of this).
- **Commented on** [PR #753](https://github.com/tobi/qmd/pull/753#issuecomment-4902197595) — scope confirmation + parallel-resolvers maintenance caution (plain comment, deliberately not a formal review).

## Testing / Research Performed

- **Daemon**: launchctl/pgrep/port checks, real MCP `initialize` handshakes before and after each restart, process start-time verification.
- **v2.6.3 shakedown**: `doctor` (fully green: better-sqlite3, sqlite-vec 0.1.9, Metal/M3, 3,486 docs on fingerprint c37385), `status`, `search` (lex phrase/negation), `vsearch`, `query` (multi-line typed query documents with `intent:` per docs/SYNTAX.md, `expand:` prefix, `--no-rerank`, `--explain`), `get` (paths, docids, `:from:count`), `multi-get` (globs, comma-lists, all pattern forms), `--format json/csv/files`, plus all four MCP tools live through the daemon.
- **Source audits**: launcher (`bin/qmd`) runtime-selection logic; pipeline constants (strong-signal 0.85/0.15, `RERANK_CANDIDATE_LIMIT=40`, RRF k=60, blend 0.75/0.60/0.40 — all unchanged); multi-get resolvers in `src/cli/qmd.ts` and `src/store.ts`; DB schema (read-only).
- **Upstream verification**: read #706 body/comments and #753 description directly before drafting; re-ran every repro block in the issue drafts verbatim before filing (caught an output-order error in one).

## Summary Statistics

- 1 dead daemon diagnosed and repaired (2 root causes: stale nvm path, then stale in-memory code)
- ~17 edits to CLAUDE.local.md across 9 sections; 3 memory files updated, 1 created
- 14+ CLAUDE.local.md claims re-validated against v2.6.3 source (5 stale, fixed; 9 confirmed)
- 3 genuine v2.6.3 defects characterized (docid multi-get, path-matching divergence, files-format field collision)
- 2 upstream issues filed, 2 upstream comments posted
- 2 of my own claims falsified by second review and corrected (memory written to prevent recurrence)

## Discoveries / Handoff Notes

- **The daemon's reported version can lie.** `serverInfo.version` and `qmd --version` read `package.json` live; a long-running process reports the new version while executing old code. Process start time is the ground truth. Now encoded in CLAUDE.local.md.
- **v2.6.3 launcher is lockfile-driven**: `package-lock.json` (ours: local, untracked) beats `bun.lock` (tracked upstream) — everything runs Node+tsx here regardless of PATH. `doctor` should always say `better-sqlite3` in this environment.
- **The LaunchAgent plist pins the nvm version path** — every Node upgrade that removes the old nvm dir will silently kill the daemon (exit status 78) until the plist is repointed. A version-independent symlink would fix this permanently (offered, not implemented).
- **Claim hygiene (from the review)**: don't infer "not indexed" from a failing command that has known bugs — verify absence with a direct positive lookup; don't call something a version change without `git log -S` receipts. Persisted as `feedback_verify_before_claiming.md`.
- **Upstream merge rhythm**: batches land roughly monthly on release days; small scoped fixes with repros+tests make the batch. #753 + #759 are well-positioned for the next one.
- **SYNTAX.md-vs-schema drift**: docs/SYNTAX.md says MCP `query` requires `searches`; the 2.6.3 schema also accepts a plain `query` string (PR #731). Not filed — docs-only nit, possibly worth a docs PR later.

## Current State

- **Daemon**: healthy, PID from 00:45:31 restart, genuine v2.6.3 code, port 8181, LaunchAgent auto-restart intact.
- **Checkout**: `main` @ e428df7 (v2.6.3), clean tree except this summary + CLAUDE.local.md (gitignored on main). `dist/` fresh (built 00:39).
- **Index**: user ran `qmd update && qmd cleanup && qmd embed` — fully current.
- **Branch topology**: `dev` holds 24 unmerged commits (mostly docs/session summaries); main has 35 commits dev lacks. Reconciliation pending.

## Cloud Routine Repurposed

The PR-watch cloud routine (`trig_01WbeXf222N9L9RADtsEBW4p`) was repurposed at the end of the session rather than retired:

- **Renamed**: `qmd-watch-716-718-719` → `qmd-watch-753-759-760`
- **New targets**: PR #753 (any movement — not ours) and issues #759/#760 (non-`rymalia` activity — ours)
- **New schedule**: twice daily at 9am & 9pm PDT (`0 4,16 * * *` UTC, was every 3 hours) — note these drift to 8am/8pm PST when DST ends in November
- Same mechanics otherwise: read-only public REST API checks, one proactive push on notable activity, dashboard report every run
- Manage at: https://claude.ai/code/routines/trig_01WbeXf222N9L9RADtsEBW4p

## Unfinished Work

- **Revert plugin marketplace pointer** to `github:tobi/qmd` when done iterating locally (unblocked since #718/#719 merged).
- **Reconcile `dev` branch** — rebase/cherry-pick its 24 commits onto main, or retire it.
- **Optional docs PR upstream**: SYNTAX.md + README still say MCP `searches` is required (stale since PR #731 added the `query` param).
- **Optional**: version-independent symlink for the LaunchAgent binary path to survive future Node upgrades.
- **Watch upstream**: #753 merge would flip the docid matrix rows; #759/#760 responses may need follow-up (the repurposed cloud routine now covers this automatically).
