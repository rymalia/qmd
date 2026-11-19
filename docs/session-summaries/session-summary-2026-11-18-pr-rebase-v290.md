---
session_id: e9083871-d327-4bff-81c7-97f65b2b8374
date: 2026-11-18
time: "4:02 AM PST – 9:58 PM PST"
project: qmd
branch: dev
related_pr: 1091
---

# Session Summary: rebasing open upstream PRs onto v2.9.0

## Overview

After `git fetch upstream` brought in qmd **v2.9.0** (`upstream/main` = `aa247d0`), the three live upstream PRs (#1090, #1091, #1094) were rebased onto it, verified locally and by Codex, and pushed by the user. #1091 gained a REST `searches` cap and a rewritten description, and #1092 was closed because upstream now covers it.

## Key Decisions Made

| Decision | Rationale |
|----------|-----------|
| **Rebase each branch in its existing worktree** (`../qmd-pr-*`) | Each PR branch was already checked out in a dedicated worktree, so `dev` in the main checkout stayed untouched |
| **Resolve the #1090 conflict by keeping both sides** | Upstream's query handler gained a `ctx` argument (`signal: ctx.mcpReq.signal` for cancellation); the PR adds `chunkStrategy`. Both are needed, and the call site passes both to `store.search()` |
| **Override `min-release-age=3` once, into a real `node_modules` in one worktree only** | v2.9.0 pins `oxlint`/`@oxlint/plugins@1.94.0` (published 2026-11-16), which the user's `~/.npmrc` guard blocked. The user chose a one-time override; installing into `qmd-pr-mcp-chunk-strategy` only kept the shared `qmd/node_modules` (used by `dev`) on the old versions |
| **Keep the REST `searches` cap in #1091** despite overlapping #1002 | Codex found REST `/query` and `/search` accept unbounded `searches`. The PR description had deliberately left that to #1002, which was missed before the cap was added. After that surfaced, the user chose to keep the cap: #1002 has been idle since 2026-10-08 and v2.9.0 shipped without it. The description says the cap is a separable commit (`a8d403e`) |
| **One shared `MAX_SEARCHES_PER_REQUEST = 10` constant** | Replaces the MCP schema's literal `.max(10)` and backs the REST check, so the two paths can't drift apart again |
| **Close #1092 rather than rebase it** | Upstream `ab589cb` replaced the `limitBinds` checks with peak-RSS (`VmHWM`) measurement via `peakRssBytes()` and moved CI's Bun pin to 1.4.4. That is the follow-up #1092 itself proposed |
| **The user posts everything to GitHub** | Per the global git rules, Claude drafted the PR description and closing comment to files; the user ran `git push`, `gh pr edit` and `gh pr close` |

## Changes Made

| Change | Detail |
|--------|--------|
| **#1091 rebased** | `fix/http-body-limit`: `e3b401d` → `0fb9f80` (no conflicts) |
| **#1094 rebased** | `docs/chunk-strategy-default-comments`: `db9abb8` → `3f69708` (no conflicts) |
| **#1090 rebased** | `feat/mcp-chunk-strategy`: `64bcc79` → `cd897ed`, with one conflict in the `src/mcp/server.ts` query handler signature. Then amended to `81da6bc` (CHANGELOG fix below) |
| **CHANGELOG entries moved back under `[Unreleased]`** (#1090, #1091) | The rebase had put both PRs' entries inside the released `## [2.9.0]` section, because upstream renamed `[Unreleased]` to `[2.9.0]` and git placed the entries by their surrounding lines |
| **REST `searches` cap** (#1091, new commit `a8d403e`) | `MAX_SEARCHES_PER_REQUEST = 10` in `src/mcp/server.ts`, used by the MCP zod schema and a REST check returning `400 {"error":"Invalid field: searches (maximum 10 items)"}`. New test "REST /query and /search reject more than 10 searches, matching MCP query" in `test/mcp.test.ts`. CHANGELOG sentence added |
| **Pushes** (by the user) | All three branches force-pushed to `origin` with `--force-with-lease -u`. GitHub reported each PR mergeable |
| **#1091 description rewritten** | Drafted, then applied by the user with `gh pr edit --body-file`. Live body confirmed identical to the draft apart from a trailing newline |
| **#1092 closed** | Closing comment drafted to a file; the user ran `gh pr close 1092 --comment …` |
| **Worktree `node_modules`** | `qmd-pr-mcp-chunk-strategy/node_modules` is now a real v2.9.0 install. `qmd-pr-http-body-limit` and `qmd-pr-chunk-default-comments` are symlinks to `/Users/rymalia/projects/qmd/node_modules` again |

## Testing / Research Performed

- **Typecheck** (`npm run test:types`) on all three rebased branches: clean.
- **MCP test files** (`CI=true vitest run test/mcp*.test.ts`): #1090 192 passed / 21 skipped; #1091 194 passed / 21 skipped before the cap and 195 / 21 after. The 21 skips are upstream's in-process LLM tests, which are off under `CI`.
- **The PRs' own tests confirmed to run:** both #1090 tests and all #1091 tests (body caps, chunked upload, collection caps, `searches` cap) showed ✓ in verbose runs with `CI=true`.
- **Proof the `searches` test checks the cap:** with the REST check removed, the new test failed (`expected 200 to be 400`); with it restored it passed. `server.ts` was confirmed identical to the committed version afterwards (`cmp`).
- **Full verification on #1091 head `a8d403e`, for the PR description:** `npx tsc --noEmit -p tsconfig.json` clean; `oxlint` 0 diagnostics across 135 files; `vitest run test/mcp.test.ts` 77 passed (Node 24.22.1); `bun test … test/mcp.test.ts` 77 pass (Bun 1.4.4); full `vitest run test/` 1847 passed / 1 skipped across 79 files.
- **`npm audit --omit=dev`** on the v2.9.0 dependencies: 0 vulnerabilities.
- **Codex reviews** (`codex exec`, `gpt-5.6-luna`, high, read-only), 3 runs:
  - #1090 rebase: SHIP, no findings.
  - #1091 rebase: FIX. The Major finding (REST `searches` uncapped) was confirmed: it already exists on `upstream/main`, so it wasn't introduced by the rebase. The Minor finding (new tests skipped under `CI`) was **disproved**: the tests sit in the `"MCP HTTP Transport — 2026-07-28 protocol"` block at line 1254, which has no `skipIf`.
  - The `searches`-cap commit: SHIP, no findings.
- **Merge checks** with `git merge-tree` of #1091 against open PRs #1002, #1036, #1047, #1057, #1059 and #1085, and against #1090. Results are recorded in the #1091 description.
- **CI investigation:** the single red jobs on #1091 (Node 23 ubuntu) and #1094 (Node 22 ubuntu) were the vitest runner error `[vitest-worker]: Timeout calling "onTaskUpdate"`, with all 79 test files passing. #1090 was 10/10 green.

## Summary Statistics

- PRs rebased: 3 (#1090, #1091, #1094). PRs closed: 1 (#1092).
- Rebase conflicts resolved: 1 code conflict (`src/mcp/server.ts`), plus 2 misplaced CHANGELOG entries fixed after a clean merge.
- New commits: 1 (`a8d403e`); amended commits: 1 (`81da6bc`).
- Codex review passes: 3, which produced 2 findings (1 confirmed and fixed, 1 disproved).

## Discoveries / Handoff Notes

- **Rebasing across a release misplaces CHANGELOG entries without any conflict.** When upstream renames `[Unreleased]` to a version, git puts your entry inside the released section. After any rebase that crosses a release, check `git diff upstream/main -- CHANGELOG.md` by eye. Moving entries back under an empty `[Unreleased]` also makes any two PRs conflict on that line (#1090 and #1091 now conflict only in `CHANGELOG.md`).
- **The PR worktrees symlink `node_modules`, and `npm install` deletes those symlinks** (`warn reify Removing non-directory …/node_modules`). The shared target is untouched, but remove the link and install into a real directory on purpose, or point the link somewhere else.
- **In npm 12, a project-scoped install rejects `--allow-scripts`** (`EALLOWSCRIPTS`). The repo's `.npmrc` already allows `better-sqlite3`, `esbuild`, `node-llama-cpp` and the tree-sitter grammars, so a plain `npm install` works. The `--allow-scripts` flag in `CLAUDE.local.md` applies only to the global `-g` install.
- **`min-release-age=3` in `~/.npmrc` blocks installing a fresh upstream release** until all its pinned dependencies are three days old. v2.9.0's `oxlint@1.94.0` was the blocker.
- **`npx oxlint` printed only npm notices.** Use `./node_modules/.bin/oxlint` (`--format=json` shows the diagnostics count).
- **Check a PR's own description before acting on a review finding about scope.** The decision to leave `searches` to #1002 lived only there, so Codex, reading only the code, flagged it as a gap.
- **GitHub only serves job logs once the whole workflow run finishes.** Before that, `gh run view --log` returns nothing; the job's `steps` and its check-run annotations show which step failed.
- **#1002 (ParkerRex) now conflicts with #1091** in the `/query` handler's `searches` validation, because of `a8d403e`. #1091 merged cleanly with #1002 before that commit.

## Current State

- `dev` in `/Users/rymalia/projects/qmd`: 43 ahead / 25 behind `upstream/main`, working tree clean. This session made no commits on `dev` before writing this summary.
- PR worktrees are clean. Each branch tracks `origin/<branch>`, and the user pushed all three.
- The published global install is still `@tobilu/qmd@2.8.3`; v2.9.0 is available.
- The LaunchAgent plist still points at the versioned nvm Node path; see the memory note on the nvm timebomb.
- Scratchpad files (session-scoped, may be cleared): `pr1091-body-new.md` and `pr1092-close-comment.md`, both already applied on GitHub.

## Issues & PRs

- https://github.com/tobi/qmd/pull/1090: rebased, `81da6bc`, CI 10/10
- https://github.com/tobi/qmd/pull/1091: rebased, `searches` cap added, description updated, `a8d403e`, CI 9/10 (runner flake)
- https://github.com/tobi/qmd/pull/1094: rebased, `3f69708`, CI 9/10 (runner flake)
- https://github.com/tobi/qmd/pull/1092: closed, replaced by upstream `ab589cb`
- https://github.com/tobi/qmd/pull/1002: someone else's open PR; now conflicts with #1091

### Unfinished Work

- Get the red Node ubuntu jobs on #1091 and #1094 re-run (maintainer re-run or an empty commit).
- Optional: `git worktree remove ../qmd-pr-bun-heap-limit-guard`, now that #1092 is closed.
- Optional: upgrade the global install to v2.9.0 (`npm install -g --allow-scripts=… @tobilu/qmd@latest`), then restart the daemon via `launchctl`. While `oxlint@1.94.0` is under three days old, the `min-release-age` guard may block this until about 2026-11-19 04:30 PST.
- Optional: update `CLAUDE.local.md`'s `--allow-scripts` note to say it applies to global installs only.
