---
session_id: 117b535c-fa88-47fe-a99d-c487c349d5ba
date: 2026-11-01
time: "05:11 PM PST – 05:15 PM PST"
project: qmd
branch: dev
---

# Session Summary: Upstream PR Status Check

## Overview

A short, read-only session checking the four open PRs on `tobi/qmd`: status, CI, reviews, and mergeability. None has any maintainer activity yet. The only red check (Bun ubuntu-latest) on three of them comes from the Bun 1.4.4 SQLite heap-budget tests that PR #1092 fixes.

## Key Decisions Made

| Decision | Rationale |
|----------|-----------|
| **Treat the Bun (ubuntu) failures as not caused by the PRs** | The failure logs for #1090 and #1091 show only the two heap-budget tests. Bun macOS and all Node jobs pass on every PR. |
| **Recommend merging #1092 first** | It fixes the test failure on `main`, so re-running CI on #1090/#1091/#1094 after it merges should turn them green. |
| **No action taken on GitHub** | The user asked for a status check only. Comments linking #1090/#1091 to #1092 were offered as an optional next step and not posted. |

## Changes Made

| Change | Detail |
|--------|--------|
| **Session summary** | This file is the only file written. No code, commits, or GitHub writes. |

## Research Performed

- `gh search prs --author=@me --repo tobi/qmd`: 13 PRs total. 4 open, 7 merged, 2 closed (#794/#795, relanded earlier as #832/#831).
- `gh pr view` + `gh pr checks` on each open PR (mergeability, merge state, review decision, checks, comments, reviews).
- `gh run view --log-failed` on Bun ubuntu runs `40256094453` (#1090) and `40256170362` (#1091) to confirm which tests fail.
- Upstream activity: `gh api repos/tobi/qmd/commits` and `gh pr list --state merged`.

## Summary Statistics

- Open PRs checked: 4
- CI runs whose failure logs were inspected: 2
- PRs with maintainer reviews/comments: 0

## Discoveries / Handoff Notes

| PR | Title | Size | CI | Merge state |
|---|---|---|---|---|
| [#1092](https://github.com/tobi/qmd/pull/1092) | test: expect the SQLite heap limit not to bind under Bun 1.4.4+ | +27/−10 | all green | CLEAN |
| [#1090](https://github.com/tobi/qmd/pull/1090) | feat(mcp): expose chunkStrategy on the MCP query tool and HTTP /query | +135/−2 | Bun ubuntu fail | UNSTABLE |
| [#1091](https://github.com/tobi/qmd/pull/1091) | fix(mcp): bound HTTP request bodies and per-request collection lists | +227/−11 | Bun ubuntu fail | UNSTABLE |
| [#1094](https://github.com/tobi/qmd/pull/1094) | docs: correct the chunkStrategy default in two comments | +2/−2 | Bun ubuntu fail | UNSTABLE |

- The two failing tests are `Embedding batching > legacy fingerprint adoption reads one copy of a body shared by 200 active paths within a 16 MiB SQLite budget` and `doctor vector sampling > the doctor check verifies large documents with duplicate paths within a 32 MiB SQLite budget`.
- All four PRs show `MERGEABLE`, with no conflicts against current `main`.
- Upstream has been quiet since **2026-10-15** (last merge: #1072, "Type-check the whole project … build with TypeScript 7"). The batch before that was brettdavies' stacked #1021–#1025 on 2026-10-06. All four PRs were opened on 2026-10-26, after the last merge.
- #1094 already has a comment from the user (2026-10-28) explaining that the Bun failure is the heap-budget issue. #1090 and #1091 have no comments.

## Unfinished Work

- Optional: comment on #1090 and #1091 pointing to #1092 as the fix for their red Bun (ubuntu) check, the same way #1094 already does.
- After #1092 merges, re-run CI on #1090/#1091/#1094. If upstream merges another batch first, check #1090/#1091 for conflicts, since both touch the MCP HTTP server.
