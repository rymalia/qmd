---
date: 2026-04-08
time: "Apr 7 1:57 PM PDT – Apr 8 12:13 AM PDT"
resumed: "1:57 PM PDT"
project: qmd
branch: dev
related_pr: 533
---

# Session Summary: Stale Index Wild Goose Chase, handleize Deep Dive, and Bench Discoverability Report

## Overview

Resumed the v2.1.0 testing session to investigate a report of stale documents in the QMD index. What started as a scary finding (39% of the index appeared stale) turned into a deep dive on the `handelize()` path normalization function, ultimately proving the index was healthy. Then pivoted to a discoverability assessment of the new `qmd bench` feature, producing a comprehensive report documenting the gap between the feature's excellent implementation and its near-zero documentation.

## Key Decisions Made

- **Investigated before assuming the worst**: The initial audit showed 1,172 of 3,032 documents appeared stale. Rather than immediately filing a bug, we traced through the code and discovered `handelize()` path normalization was creating a false positive in our audit methodology.
- **Corrected the audit methodology**: Re-ran the audit using `handelize()` normalization to match what QMD actually does internally. Reduced the "stale" count from 1,172 → 3 → 0.
- **Assessed bench from user perspective only**: Deliberately avoided using source code knowledge to evaluate bench discoverability. Read only README, `--help`, and MCP tool discovery — the same surfaces a new user would see.
- **Wrote a formal report**: The bench findings warranted a standalone document rather than inline notes, since it feeds into the broader README/help revamp plans.

## Changes Made

| Change | Detail |
|--------|--------|
| **Bench discoverability report** | `docs/bench-discoverability-report-2026-04-07.md` — full assessment with recommendations for README, --help, error messages, output, and fixture docs |
| **Comment on #520** | Added `handelize()` `c++` → `c` collision example to the existing issue about handleized paths in search results |

## The Wild Goose Chase: Stale Index Investigation

### The Trigger

Another session reported a stale document: `tweet/docs/session-summary-2026-03-04-gemini-skill-evaluation.md` scored 0.93 relevance but allegedly no longer existed on disk. The user confirmed they run `qmd update`, `qmd embed`, and `qmd cleanup` regularly.

### Phase 1: The Alarming Audit

Queried the SQLite database for all 3,032 active documents and checked each against the filesystem by reconstructing `collection_path + document_path`. Results: **1,172 documents (39%) appeared stale.** Some collections were 80-90% stale:

| Collection | Stale % |
|-----------|---------|
| mac-whisper-speedtest | 92% |
| wizard-data-dnd-srd-5.2-markdown | 90% |
| mlx-audio | 82% |
| pipecat | 82% |

### Phase 2: The Realization

Spot-checking revealed the files weren't deleted — they were **renamed**. The filesystem had `APPLE_SILICON_OPTIMIZATIONS.md` and `feat_model-handling-issues.md`, but the DB stored `apple-silicon-optimizations.md` and `feat-model-handling-issues.md`. The `handelize()` function normalizes paths: lowercases, converts underscores to hyphens, strips non-alphanumeric characters.

Key test:
```
handelize('APPLE_SILICON_OPTIMIZATIONS.md') → 'apple-silicon-optimizations.md'
handelize('apple-silicon-optimizations.md') → 'apple-silicon-optimizations.md'
MATCH: true
```

Both the old and new filenames produce the same handleized path. QMD's `reindexCollection` reads content from the real filesystem path, hashes it, and stores it under the handleized key. The content is up-to-date — our audit was checking the wrong thing.

### Phase 3: The Corrected Audit

Re-ran the audit using `handelize()` to normalize filesystem paths before comparing against DB paths. Results: **1,169 false positives, 3 true mismatches.** The 3 mismatches were in `pipecat-docs/client/c++/` — where `handelize` strips `++` from the directory name, producing `client/c/`. The files exist on disk at `client/c++/`, the DB stores them at `client/c/`, and the content is correct.

**Final count: 0 truly stale documents out of 3,032.**

### The `c++` Edge Case

`handelize('client/c++/api-reference.mdx')` → `client/c/api-reference.mdx`. The `++` is stripped entirely. This is the same class of information loss documented in issue #100 (symbol-only filenames) and #520 (handleized paths in search results). We commented on #520 with this concrete collision example — if a `client/c/` directory were added alongside `client/c++/`, documents would shadow each other.

## Bench Feature Assessment

### What We Found

Approached `qmd bench` as a new user with no source code knowledge:

- **README** (945 lines): Zero mentions of bench
- **`--help`**: One line, no examples, no fixture format
- **MCP**: Not exposed
- **Error on missing args**: Points to `src/bench/fixtures/example.json` — a source code path unreachable to npm users

### The Silent Failure

Ran bench without the eval-docs collection indexed. It executed 40 queries across 4 backends, spent ~2 minutes loading models, and produced a wall of zeros with no warning that the collection didn't exist. No error, no suggestion.

### The Real Results

After proper setup (`collection add test/eval-docs + embed`), bench produced excellent results demonstrating why hybrid search exists:

| Backend | P@k | Recall | MRR | F1 | Avg Latency |
|---------|-----|--------|-----|-----|-------------|
| bm25 | 0.500 | 0.500 | 0.500 | 0.500 | 72ms |
| vector | 0.700 | 0.700 | 0.700 | 0.700 | 86ms |
| hybrid | **1.000** | **1.000** | **1.000** | **1.000** | 1,377ms |
| full | **1.000** | **1.000** | **1.000** | **1.000** | 750ms |

BM25 fails on synonyms/concepts (5/10 queries). Vector fails on exact phrases/keywords (3/10 queries). Hybrid fusion eliminates both failure modes — perfect recall.

### The Report

Produced `docs/bench-discoverability-report-2026-04-07.md` — a comprehensive document mapping the full user journey (discovery → setup → failure → interpretation), connecting each issue to patterns from the v2.0.1 onboarding feedback, and providing concrete recommendations with copy-pasteable examples for README, --help, error messages, and output formatting.

## Research Performed

- **Stale index audit**: Queried all 3,032 active documents against filesystem, corrected for `handelize()` normalization
- **Code tracing**: `reindexCollection()`, `handelize()`, `deactivateDocument()`, `searchFTS()`, `updateCollections()` — verified deactivation logic is correct
- **Bench discoverability**: Evaluated README (945 lines), --help output, MCP tool list, error messages, and fixture accessibility
- **Bench execution**: Ran full benchmark (10 queries × 4 backends) both without and with eval-docs collection
- **Upstream issues**: Reviewed #100, #513, #520 for handelize-related prior art

## Summary Statistics

- **Documents audited**: 3,032 (full index)
- **False positive rate of naive audit**: 99.7% (1,169/1,172)
- **Truly stale documents**: 0
- **Benchmark queries executed**: 80 (40 without collection, 40 with)
- **Files created**: 1 report (`bench-discoverability-report-2026-04-07.md`)
- **Upstream comments**: 1 (on #520)

## Discoveries / Handoff Notes

### handelize() is Both Resilient and Lossy

`handelize()` makes QMD resilient to file renames (case changes, underscore/hyphen swaps) by normalizing all paths to a canonical form. But it also creates information loss:
- `SCREAMING_CASE.md` and `screaming-case.md` collapse to the same key (intended, useful)
- `client/c++/` and `client/c/` collapse to the same key (unintended, lossy)
- Audit scripts that compare DB paths against raw filesystem paths will massively overcount staleness

PR #513 (open) adds a `source_path` column to preserve the original filesystem path alongside the handleized one. PR #100 (open) adds codepoint encoding for symbol-only filenames. Both would help.

### Naive Filesystem Audits Against QMD Are Unreliable

Any script that does `if [ ! -f "$basepath/$db_path" ]` will report false positives because `db_path` is handleized. The correct approach is to run the filesystem paths through `handelize()` before comparing — which is what `reindexCollection` already does internally.

### The Bench Feature Is a Selling Point Hiding in the Changelog

The benchmark results make the strongest possible case for QMD's hybrid architecture — neither BM25 nor vector alone breaks 0.700, but hybrid hits 1.000. This should be front-and-center in the README, not buried in a changelog entry.

## Issues & PRs

### Commented On

- **[#520](https://github.com/tobi/qmd/issues/520)** — Added `c++` → `c` collision example from our audit as a concrete case of `handelize()` information loss

### Referenced

- **[#100](https://github.com/tobi/qmd/issues/100)** — PR for codepoint encoding of symbol-only filenames in `handelize()`
- **[#513](https://github.com/tobi/qmd/pull/513)** — PR adding `source_path` column to preserve original filesystem paths
- **[#533](https://github.com/tobi/qmd/pull/533)** — Our PR from earlier session (still open, CI flake noted)

## Unfinished Work

- **README revamp**: The bench report and v2.0.1 onboarding feedback together form a comprehensive brief for overhauling the README and --help content. The specific recommendations in the bench report are ready to implement.
- **Bench guardrails**: Collection validation, all-zero warning, `--example` command, and metric legend — all straightforward changes documented in the report.
- **Custom benchmark fixture**: Could create a fixture targeting the real 3,032-doc index (not just eval-docs) to test search quality on actual data at scale.
