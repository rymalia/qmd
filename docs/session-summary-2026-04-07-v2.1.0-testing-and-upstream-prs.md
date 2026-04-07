---
date: 2026-04-07
time: "11:19 AM PDT – 1:35 PM PDT"
project: qmd
branch: dev
related_pr: 533
---

# Session Summary: QMD v2.1.0 Testing, Bug Discovery, and Upstream PRs

## Overview

Comprehensive testing of qmd v2.1.0 after upgrading from v2.0.1. Discovered two bugs in the upstream codebase — one in CLI JSON output (`line` field missing) and one in multi-collection search (CTE candidate starvation). Submitted PR #533 to fix the JSON `line` field, posted detailed comments on 4 existing issues/PRs to advance stalled upstream fixes, and performed 30+ edge case tests covering CLI, MCP, search pipeline, and security.

## Key Decisions Made

- **Fix locally, then PR upstream**: Applied the `line` field fix to our local build first to verify it worked, then created a clean branch from main for the PR. This kept our dev branch unblocked while contributing upstream.
- **Comment on existing issues rather than filing new ones**: For the multi-collection bug (#217) and the vec/hyde hyphen validation (#383), existing issues already described the problems well. We added confirmation, root cause analysis, and v2.1.0 reproduction steps instead of creating duplicates.
- **Credit the original author on PR #241**: Rather than picking up the stalled multi-collection fix ourselves, we commented on PR #241 with specific guidance on what needed fixing, giving @ambicuity the opportunity to finish their work.
- **Don't file issues for `-n 0`/`-n -1`**: These edge cases (zero/negative limit returning default results) are minor and unlikely to affect real usage. Not worth the noise.

## Changes Made

| Change | Detail |
|--------|--------|
| **Fix: `line` in JSON output** | `src/cli/qmd.ts:1935-1944` — Extract full `SnippetResult` from `extractSnippet()` and spread `line` into JSON output object. 3-line change. |
| **PR #533 submitted** | `fix/json-line-field` branch pushed to `rymalia/qmd`, PR opened against `tobi/qmd` main |
| **Comment on #383** | Confirmed vec/hyde hyphen validation bug persists in v2.1.0, referenced PR #384 as the fix, linked duplicate issues #390 and #414 |
| **Comment on #463** | Clarified that the merged lex/FTS5 fix does not address #383/#384/#390/#414 (vec/hyde validation is a different code path) |
| **Comment on #217** | Confirmed multi-collection CTE starvation in v2.1.0 with reproduction steps, identified root cause at `store.ts:2941`, referenced PR #241 |
| **Comment on #241** | Encouraged reopening the stalled PR with specific guidance on the two issues Copilot flagged |

## Testing / Research Performed

### v2.1.0 Feature Verification

All new v2.1.0 features tested and working:

- **AST chunking**: `qmd status` shows 6 language grammars active; `--chunk-strategy auto` returns results
- **`--no-rerank` flag**: ~4s vs ~12s for full pipeline; correctly skips reranking step
- **`--explain` flag**: Full RRF score traces with per-query contributions, reranker scores, blended scores
- **`qmd bench`**: Command available; example fixture found at `src/bench/fixtures/example.json` (requires `eval-docs` collection to be indexed to run)
- **`get :line` syntax**: Line-slice retrieval working
- **Per-collection model config**: `models:` section documented in changelog; `qmd status` shows model URIs
- **MCP `rerank: false`**: Working via MCP query tool
- **MCP `intent` parameter**: Working — disambiguates results
- **Strong-signal bypass**: Correctly skips expansion when BM25 score >= 0.85

### Edge Case / Stress Testing (30+ tests)

| Category | Tests | Findings |
|----------|-------|----------|
| **Empty/invalid input** | Empty query, 10K char query, `-n 0`, `-n -1` | Empty query: clean error. Long query: graceful empty. `-n 0`: returns default (minor). |
| **Unicode/emoji** | Emoji query `🔥🎯🐘` | Found changelog entry about emoji filenames — correct and clever |
| **Security** | SQL injection (`SELECT * DROP TABLE`, Bobby Tables variant) | FTS5 parameterized queries prevent injection. DB intact. |
| **Collection filtering** | Nonexistent collection, multi-collection `-c a -c b` | Nonexistent: clean error. **Multi-collection: BUG — empty results due to CTE starvation** |
| **Search features** | Quoted phrases, negation, multi-line query docs | All working correctly |
| **MCP tools** | Status, query, empty queries, high minScore | All graceful — empty results or correct data |
| **Output formats** | `--json`, `--explain`, `--files`, `--md` | All working; `line` field now present in JSON after our fix |

### Code Path Analysis

Traced the `line` field through three layers to find why PR #506 was dead code:
1. `extractSnippet()` in `store.ts:3751` — returns `SnippetResult` with `line: number`
2. `searchResultsToJson()` in `formatter.ts:119` — includes `line` (PR #506's fix) but **has zero callers**
3. `outputResults()` in `qmd.ts:1930` — the actual CLI JSON path, which bypassed the formatter

Also traced the multi-collection bug through `searchFTS` → CTE `ftsLimit` → `filterByCollections` to identify the exact line (`store.ts:2941`) where the 10× headroom is missing.

## Summary Statistics

- **Tests run**: 774 unit tests (all passing) + 30+ manual edge case tests
- **Bugs found**: 2 significant (JSON `line` field, multi-collection search), 2 minor (`-n 0`/`-n -1`)
- **PRs submitted**: 1 (PR #533)
- **Upstream comments**: 4 (on #383, #463, #217, #241)
- **Files modified**: 1 (`src/cli/qmd.ts` — 3 lines changed)
- **Source files traced**: `store.ts`, `cli/qmd.ts`, `cli/formatter.ts`, `mcp/server.ts`

## Discoveries / Insights

### The Orphaned Formatter Module

The `formatter.ts` module contains a full suite of shared formatting functions (`searchResultsToJson`, `formatSearchResults`, etc.) that are exported, imported in `qmd.ts`, but **never called** for search results. The CLI's `outputResults()` has its own inline formatting that evolved independently. This is why PR #506's fix to `searchResultsToJson()` was completely dead code — it fixed a function with zero callers. This is a textbook case of abstraction divergence in fast-moving projects.

### PR #463's Over-Claiming

PR #463 (merged in v2.1.0) claimed `Fixes #414, #383, #384, #390, #417` but only addressed FTS5 lex parsing — a completely different code path from the vec/hyde `validateSemanticQuery` issues (#383, #384, #390, #414). The `/-\w/` regex at `store.ts:2910` is untouched. Three separate people filed the same vec/hyde validation bug, indicating it's a real pain point. Our PR #384 remains the correct fix.

### Multi-Collection Search: A Known, Stalled Fix

The multi-collection CTE starvation bug has 4 open issues (#175, #181, #217, #233) spanning v1.0.0 to v1.0.7, all still unfixed in v2.1.0. PR #241 by @ambicuity had the correct SQL `IN` approach but was closed without merge after only a Copilot review. The two issues flagged (per-collection loop in `structuredSearch`, wrong test exit code) are straightforward to fix.

### Position-Aware Score Blending

The `--explain` output reveals QMD's scoring transparency: for rank 1 results, the blend is 75% RRF / 25% reranker; rank 4-10 shifts to 60/40; rank 11+ shifts to 40/60. The reranker's influence increases for lower-ranked results where RRF signal is weaker. This is visible in the `explain.rrf.weight` field and explains why `--no-rerank` produces noticeably different orderings for results beyond position 3.

### BM25 Fixes Are the Invisible Win

v2.1.0's 6 BM25/embedding fixes (correct FTS field weights #462, hyphenated tokens #463, underscore preservation #404, CTE query planner fix, explicit embed context #500, dimension mismatch detection #501) compound to improve every query. Less flashy than AST chunking but more impactful day-to-day.

## Issues & PRs

### PRs Submitted

- **[PR #533](https://github.com/tobi/qmd/pull/533)** — `fix: include line in CLI --json search output` (new, submitted this session)
- **[PR #384](https://github.com/tobi/qmd/pull/384)** — `fix: allow hyphenated words in vec/hyde queries` (existing, now with fresh context)

### Issues Commented On

- **[#383](https://github.com/tobi/qmd/issues/383)** — vec/hyde hyphen validation still broken in v2.1.0
- **[#217](https://github.com/tobi/qmd/issues/217)** — Multi-collection CTE starvation confirmed in v2.1.0

### PRs Commented On

- **[#463](https://github.com/tobi/qmd/pull/463)** — Clarified scope: fixes lex parsing only, not vec/hyde validation
- **[#241](https://github.com/tobi/qmd/pull/241)** — Encouraged reopening with specific guidance on remaining issues

### CI Note

PR #533's CI has a `Bun (ubuntu-latest)` failure in `createStore throws without explicit path in test mode` — a pre-existing Bun test isolation flake (module state leak between test files). Unrelated to our 3-line change. The `store.helpers.unit.test.ts` version of the same test calls `_resetProductionModeForTesting()` but `store.test.ts` doesn't.

## Unfinished Work

- **`qmd bench` not run**: Requires `eval-docs` collection to be indexed first (`qmd collection add test/eval-docs --name eval-docs && qmd embed`)
- **Multi-collection fix PR**: If PR #241 stays dormant, we could pick up the SQL `IN` fix ourselves — the approach is validated, just needs the `structuredSearch` loop and test assertion fixed
- **PR #384 review**: Still open and waiting for maintainer review; the fresh comments on #383 and #463 should provide context
