# Session Summary: 2026-03-12 - Semantic Query Hyphen Validation Fix

**Date**: 2026-03-12
**Duration**: ~45 minutes
**Branch**: `fix/semantic-query-hyphen-validation` (from `main`)

## TL;DR

Started by reconnecting the QMD MCP server and running a demo search. During a more substantial query investigating the Feb 24 HTTP-vs-stdio decision, a hyde sub-query containing "long-lived" was rejected as negation syntax. Traced the bug to an overly broad regex in `validateSemanticQuery`, wrote a one-line fix, added 14 unit tests, filed issue #383, and opened PR #384.

## Key Decisions Made

| Decision | Rationale |
|----------|-----------|
| **Fix regex to `/(^\|\s)-[\w"]/`** | Anchors negation detection to token boundaries (start of string or after whitespace), letting mid-word hyphens through |
| **Consolidate two regex checks into one** | `/-\w/` and `/-"/` merged into single `/(^\|\s)-[\w"]/` — cleaner and handles both cases |
| **Branch from `main`, not `dev`** | Clean isolated fix targeting `main` via PR, avoids entangling with unrelated `dev` work |

## Changes Made

| Change | Detail |
|--------|--------|
| **`src/store.ts:2601`** | Replaced `/-\w/.test(query) \|\| /-"/.test(query)` with `/(^\|\s)-[\w"]/.test(query)` — one-line regex fix |
| **`test/structured-search.test.ts`** | Expanded `validateSemanticQuery` tests from 3 to 14: 6 negation rejection cases, 10+ hyphenated word acceptance patterns, multi-hyphen phrases, short terms, bare hyphen, hyde with hyphens |
| **`docs/issue-draft-semantic-query-hyphen-validation.md`** | Full issue writeup with root cause analysis, scope of impact, and proposed fix |

## The Bug

`validateSemanticQuery` used `/-\w/` to detect negation syntax in vec/hyde queries. This regex matches any hyphen followed by a word character **anywhere** in the string — not just at token boundaries where negation actually occurs. Result: 19 out of 20 common hyphenated word patterns were false positives.

Affected patterns include kebab-case identifiers (`better-sqlite3`, `node-llama-cpp`), compound adjectives (`real-time`, `multi-client`, `self-hosted`), prepositional compounds (`in-memory`, `write-ahead`, `copy-on-write`), and multi-hyphen phrases (`state-of-the-art`, `end-to-end`).

The lex query parser (`buildFTS5Query`) already handles this correctly — it tokenizes by whitespace first, then checks for `-` at token start. The validator was a quick pre-check regex that skipped the tokenization step.

## Testing Performed

- Built a 33-case test matrix comparing current vs proposed regex (25 non-negation, 5 negation, 3 edge cases)
- Current regex: 19/20 non-negation queries falsely rejected
- Fixed regex: 32/33 cases correct (only `--verbose` edge case debatable — not real negation syntax)
- All 709 existing tests pass after fix
- 14 new unit tests pass
- End-to-end CLI test: `qmd query 'vec: long-lived daemon'` passes validation and reaches search phase

## Issues & PRs

- **Issue**: [#383 — vec/hyde queries reject hyphenated words as negation syntax](https://github.com/tobi/qmd/issues/383)
- **PR**: [#384 — fix: allow hyphenated words in vec/hyde queries](https://github.com/tobi/qmd/pull/384)

## Summary Statistics

- 1 regex changed (1 line)
- 14 new test cases added
- 709/709 full suite passing
- 1 issue filed, 1 PR opened

## How We Got Here

Session began with a QMD MCP demo search. A follow-up query about the Feb 24 HTTP MCP decision used a hyde sub-query with the word "long-lived" — which was rejected as negation. Rather than work around it, we traced the bug, characterized it across 33 test cases, fixed it, and shipped it as a proper issue + PR. The original MCP investigation is deferred to a future session.

## Unfinished Work

- **Phase 4c MCP server inspection**: The original goal of this session — continue validating the v2.0 MCP server — was deferred in favor of the bug fix. Pick up from the Feb 24 session context.
- **`dev` branch**: Still has unrelated uncommitted changes (marketplace.json, deleted bun.lock, finetune docs). These predate this session.
