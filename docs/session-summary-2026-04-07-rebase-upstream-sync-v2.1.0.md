---
date: 2026-04-07
time: "12:35 AM PDT – 11:16 AM PDT"
project: qmd
version: v2.1.0
branch: dev
---

## Overview

Synced local dev branch with 45 upstream commits from origin/main using rebase for a clean linear history, then rebuilt and upgraded the local qmd install from v2.0.1 to v2.1.0.

## Key Decisions Made

| Decision | Rationale |
|----------|-----------|
| **Rebase instead of merge** | User wanted to preserve individual commit history from main in a linear log, avoiding merge commits that obscure upstream history |
| **Skip old merge commit during rebase** | Commit `a67c8a9` ("Merge main into dev: pick up 25 upstream commits") was a previous merge whose changes were already in main — replaying it was redundant and caused conflicts |
| **Force push with lease** | `--force-with-lease` over `--force` for safety on the solo fork branch |
| **Kill daemon last, not first** | LaunchAgent's `KeepAlive: true` auto-restarts the process, so killing first would just restart with old code during build |
| **Leave npm audit vulnerabilities alone** | All 3 high-severity findings are in dev/test transitive deps (vite, path-to-regexp, picomatch <=2.3.1), not in runtime code — fixing would create unnecessary diff noise against upstream |

## Changes Made

| Change | Detail |
|--------|--------|
| **Rebased dev onto main** | Replayed 9 dev-only commits on top of 45 new upstream commits, skipping one stale merge commit |
| **Force pushed dev** | Updated `origin/dev` with rebased history (`6b8e8c5...b2e3518`) |
| **Rebuilt local install** | `npm install && npm run build` — picked up new deps (web-tree-sitter, tree-sitter grammars, sqlite-vec 0.1.9, node-llama-cpp 3.18.1) |
| **Restarted MCP daemon** | `pkill -f "qmd.js mcp --http"` — LaunchAgent auto-restarted with v2.1.0 code |
| **Updated CLAUDE.local.md** | Version bumped from v2.0.1 to v2.1.0 |

## Educational Topics Covered

This was a teaching-heavy session. Topics discussed with the user:

- **`git merge` vs `git pull`** — pull = fetch + merge; merge alone requires a prior fetch
- **`--ff-only` as a safety guard** — prevents unexpected merge commits on mirror branches
- **Why merge (not pull) for cross-branch updates** — dev tracks origin/dev, not origin/main
- **Merge vs rebase tradeoffs** — merge preserves topology but adds merge commits; rebase gives linear history but rewrites hashes
- **What "rewrite hashes" means** — parent change → new hash, even with identical diffs
- **`--force-with-lease` vs `--force`** — lease checks for upstream changes before overwriting
- **`npm install` vs `npm run build`** — deps vs compilation; when you need each
- **LaunchAgent KeepAlive and daemon restart ordering** — why killing last is correct
- **npm audit triage** — dev-only transitive deps in a local CLI tool pose no real risk

## Testing / Research Performed

- Verified `bun.lock` conflict resolution during both merge attempt and rebase
- Confirmed `npm install` picked up 7 new packages and updated 12
- Build completed without TypeScript errors
- MCP server reconnected successfully after daemon restart

## Summary Statistics

- 45 upstream commits integrated into dev
- 7 new npm packages installed, 12 updated
- 1 file edited (CLAUDE.local.md version bump)
- 0 code changes by us — this was purely a sync and upgrade session

## Discoveries / Handoff Notes

- **QMD is now v2.1.0** with notable new features: AST-aware chunking via tree-sitter (web-tree-sitter + language grammars for Go, Python, Rust, TypeScript), sqlite-vec upgraded to 0.1.9, node-llama-cpp to 3.18.1, new benchmark tooling (`src/bench/`), and pnpm-lock.yaml added upstream
- **Old merge commits cause conflicts during rebase** — `git rebase --skip` is the correct resolution when the merge's changes are already in the target branch
- **`bun.lock` conflict will recur** on every upstream sync if upstream keeps modifying it — resolve with `git rm bun.lock` (merge) or `--skip` the old merge commit (rebase)
