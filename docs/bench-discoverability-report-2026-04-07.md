---
title: "QMD Bench: Feature Discoverability & Documentation Gap Report"
date: 2026-04-07
type: feedback
tags: [qmd, bench, benchmark, ux, documentation, discoverability, onboarding]
project: qmd
version: v2.1.0
related_issues: [520, 513, 100]
---

# QMD Bench: Feature Discoverability & Documentation Gap Report

> An assessment of the `qmd bench` command introduced in v2.1.0, evaluating the gap between the feature's actual capability and how well it communicates itself to users (human and agent). Conducted 2026-04-07 on an Apple M3, v2.1.0 (source build), 48 collections / 3032 documents.

## Executive Summary

`qmd bench` is a well-engineered search quality benchmarking tool that tests 4 backends across 5 IR metrics with clean tabular output. **The feature itself scores 9/10. Its documentation and discoverability score 1/10.** A user who doesn't read the source code cannot discover the feature exists, cannot learn how to set it up, gets no feedback when it fails, and receives no explanation of what the results mean.

This report documents the full gap with concrete examples, traces the user journey from discovery to interpretation, and provides specific recommendations for the README, `--help`, error messages, and output formatting.

---

## Table of Contents

- [The Feature: What Bench Actually Does](#the-feature-what-bench-actually-does)
- [The Gap: Discovery Through Interpretation](#the-gap-discovery-through-interpretation)
  - [Stage 1: Does This Feature Exist?](#stage-1-does-this-feature-exist)
  - [Stage 2: How Do I Set It Up?](#stage-2-how-do-i-set-it-up)
  - [Stage 3: What Happens When It Fails?](#stage-3-what-happens-when-it-fails)
  - [Stage 4: What Do the Results Mean?](#stage-4-what-do-the-results-mean)
- [Actual Benchmark Results](#actual-benchmark-results)
  - [Summary Table](#summary-table)
  - [Per-Query Analysis](#per-query-analysis)
  - [What the Results Prove](#what-the-results-prove)
- [Comparison to v2.0.1 Onboarding Feedback](#comparison-to-v201-onboarding-feedback)
- [Recommendations](#recommendations)
  - [README Additions](#readme-additions)
  - [--help Improvements](#--help-improvements)
  - [Error Messages & Guardrails](#error-messages--guardrails)
  - [Output Improvements](#output-improvements)
  - [Fixture Documentation](#fixture-documentation)
- [Appendix: Full Benchmark Output](#appendix-full-benchmark-output)

---

## The Feature: What Bench Actually Does

`qmd bench` runs a structured search quality evaluation. Given a fixture file (JSON) containing queries with known-relevant documents, it executes each query against 4 search backends and measures 5 information retrieval metrics.

**Backends tested:**

| Backend | Pipeline | LLM Required |
|---------|----------|--------------|
| `bm25` | BM25 keyword search only (FTS5) | No |
| `vector` | Vector similarity only (embeddinggemma) | Embedding model |
| `hybrid` | BM25 + vector + RRF fusion, no reranking | Embedding model |
| `full` | BM25 + vector + RRF + query expansion + LLM reranking | All 3 models |

**Metrics per query:**

| Metric | What It Measures |
|--------|-----------------|
| **P@k** (Precision at k) | Of the top-k results returned, what fraction were relevant? |
| **Recall** | Of all expected files, what fraction appeared anywhere in results? |
| **MRR** (Mean Reciprocal Rank) | How high did the first relevant result appear? (1/rank) |
| **F1** | Harmonic mean of precision and recall |
| **Latency** | Wall-clock time in milliseconds |

**Fixture format** (JSON):
```json
{
  "description": "Example benchmark fixture for QMD eval-docs",
  "version": 1,
  "collection": "eval-docs",
  "queries": [
    {
      "id": "exact-api",
      "query": "API versioning",
      "type": "exact",
      "description": "Direct keyword match in API design document",
      "expected_files": ["api-design-principles.md"],
      "expected_in_top_k": 1
    }
  ]
}
```

**Query types** supported: `exact`, `semantic`, `topical`, `cross-domain`, `alias`.

The shipped example fixture contains 10 queries across all 5 types, targeting 6 test documents in `test/eval-docs/`.

**This is a genuinely useful feature.** It lets you measure search quality objectively, compare backends, tune parameters, and regression-test after index or model changes. The implementation is clean, the output is readable, and it exercises the full pipeline.

---

## The Gap: Discovery Through Interpretation

### Stage 1: Does This Feature Exist?

**README.md** (945 lines): Zero mentions of `bench`, `benchmark`, `precision`, `recall`, `MRR`, or `F1`. A user reading the full README from top to bottom will not learn that benchmarking exists.

The README covers: Quick Start, MCP Server, SDK/Library Usage, Architecture, Score Normalization & Fusion, Requirements, Installation, Collection Management, Embedding, Context, Search Commands, Options, Output Format, Examples, Index Maintenance, Data Storage, Environment Variables, How It Works (Indexing Flow, Embedding Flow, Smart Chunking, Query Flow), and Model Configuration. Benchmarking is absent from all sections.

**`qmd --help`**: One line buried in a wall of text:
```
qmd bench <fixture.json>      - Run search quality benchmarks against a fixture file
```

No examples. No explanation of what a "fixture file" is. No mention of what backends are tested or what metrics are produced. No indication that setup is required before running.

**MCP tools**: Not exposed. The 4 MCP tools are `query`, `get`, `multi_get`, and `status`. An agent using QMD via MCP has no way to discover or invoke bench.

**Changelog** (v2.1.0): Listed as a feature:
```
- `qmd bench <fixture.json>` command for measuring search quality.
  Runs queries from a fixture file against BM25, vector, hybrid, and
  full pipeline backends. Reports precision@k, recall, MRR, and F1.
  Ships with an example fixture against the eval-docs test collection.
```

This is the most complete description of bench in any user-facing document — and it's in the changelog, which users consult for "what changed" not "how to use."

**Verdict**: A user who reads the full README and runs `qmd --help` has a ~5% chance of noticing bench exists. An agent has ~0% unless it reads the changelog.

### Stage 2: How Do I Set It Up?

Assume a user noticed the `--help` line and wants to try it. Their journey:

**Step 1: Get a fixture file.**
```
$ qmd bench
Usage: qmd bench <fixture.json> [--json] [-c collection]

Run search quality benchmarks against a fixture file.
See src/bench/fixtures/example.json for the fixture format.
```

The error message points to `src/bench/fixtures/example.json` — a **source code path**. This only resolves if you're inside the git repo:

- **Git clone user** (`cd qmd && qmd bench src/bench/fixtures/example.json`): Works.
- **npm install -g user**: The file exists deep inside `node_modules` at a path like `~/.nvm/versions/node/v24.12.0/lib/node_modules/@tobilu/qmd/src/bench/fixtures/example.json`. The user would need to run `npm root -g` to find it. The error message gives no hint about this.
- **npx user**: The file is in a temporary cache directory. Effectively unreachable.

**Step 2: Index the eval-docs collection.**

The example fixture specifies `"collection": "eval-docs"`. This collection doesn't exist by default — it targets 6 test documents in `test/eval-docs/`. Nowhere does any documentation say you need to:

```sh
qmd collection add test/eval-docs --name eval-docs
qmd embed
```

A user who skips this step (because nothing told them to do it) gets the behavior described in Stage 3.

**Step 3: Run the benchmark.**

Even if the user finds the fixture and indexes eval-docs, there's no documentation explaining:
- What the 4 backends are or how they differ
- What P@k, Recall, MRR, F1 mean
- What "good" vs "bad" scores look like
- How to create their own fixture for their own collections
- The fixture JSON schema (query types, expected_files, expected_in_top_k)

**Verdict**: The setup requires 3 undocumented steps (find fixture, index collection, embed). Only a source code reader can complete this sequence.

### Stage 3: What Happens When It Fails?

We ran bench without the eval-docs collection indexed. The result:

```
Query                     Backend  P@k    Recall  MRR    F1     ms
----------------------------------------------------------------------
exact-api                 bm25      0.00  0.00   0.00  0.00       2ms
exact-api                 vector    0.00  0.00   0.00  0.00     442ms
exact-api                 hybrid    0.00  0.00   0.00  0.00     410ms
exact-api                 full      0.00  0.00   0.00  0.00      81ms
[... 36 more rows of zeros ...]

Summary:
----------------------------------------------------------------------
  bm25     P@k= 0.000 Recall= 0.000 MRR= 0.000 F1= 0.000 Avg=0ms
  vector   P@k= 0.000 Recall= 0.000 MRR= 0.000 F1= 0.000 Avg=66ms
  hybrid   P@k= 0.000 Recall= 0.000 MRR= 0.000 F1= 0.000 Avg=2489ms
  full     P@k= 0.000 Recall= 0.000 MRR= 0.000 F1= 0.000 Avg=94ms
```

**40 queries ran** (10 queries x 4 backends). **~2 minutes of wall-clock time** loading models and searching. **Zero indication anything was wrong.** No warning that the collection doesn't exist. No suggestion to index it. Just a wall of zeros that a user might interpret as "my search engine is broken" rather than "the test collection isn't set up."

Compare this to how QMD handles other missing-collection cases:
```
$ qmd search "test" -c nonexistent
Collection not found: nonexistent
```

Clean error, immediate feedback. Bench should do the same.

**Verdict**: Silent failure. A user loses 2 minutes and gains confusion.

### Stage 4: What Do the Results Mean?

After properly indexing eval-docs, the benchmark produces meaningful results. But the output provides no interpretation:

```
Summary:
----------------------------------------------------------------------
  bm25     P@k= 0.500 Recall= 0.500 MRR= 0.500 F1= 0.500 Avg=72ms
  vector   P@k= 0.700 Recall= 0.700 MRR= 0.700 F1= 0.700 Avg=86ms
  hybrid   P@k= 1.000 Recall= 1.000 MRR= 1.000 F1= 1.000 Avg=1377ms
  full     P@k= 1.000 Recall= 1.000 MRR= 1.000 F1= 1.000 Avg=750ms
```

Questions a user has at this point:
- What do P@k, Recall, MRR, F1 mean?
- Is 0.500 bad? Is 1.000 expected?
- Why did bm25 score 0.500 — which queries did it miss?
- Why is hybrid slower than full? (Model caching from previous run)
- What should I do with this information?

None of these are answered in the output, the `--help`, or the README.

**Verdict**: The output is data without context. A search engineer knows what MRR means. A developer using QMD to index their notes does not.

---

## Actual Benchmark Results

After properly setting up the eval-docs collection (`qmd collection add test/eval-docs --name eval-docs && qmd embed`), the benchmark produces excellent results that showcase exactly why hybrid search matters.

### Summary Table

| Backend | P@k | Recall | MRR | F1 | Avg Latency |
|---------|-----|--------|-----|-----|-------------|
| **bm25** | 0.500 | 0.500 | 0.500 | 0.500 | 72ms |
| **vector** | 0.700 | 0.700 | 0.700 | 0.700 | 86ms |
| **hybrid** | 1.000 | 1.000 | 1.000 | 1.000 | 1,377ms |
| **full** | 1.000 | 1.000 | 1.000 | 1.000 | 750ms |

### Per-Query Analysis

The benchmark reveals the exact complementary blind spots of BM25 and vector search:

**BM25 failures** (5/10 queries scored 0.00):

| Query | Type | Why BM25 Failed |
|-------|------|----------------|
| `semantic-rest` ("how to structure REST endpoints") | semantic | No keyword overlap with target doc "api-design-principles.md" |
| `semantic-overfitting` ("how to prevent models from memorizing data") | semantic | Conceptual match for "overfitting" — no shared keywords |
| `topical-launch` ("what went wrong with the product launch") | topical | Topic-level match, no exact terms |
| `cross-domain-consistency` ("consistency vs availability tradeoffs") | cross-domain | CAP theorem by concept, not by name |
| `alias-remote` ("working from home guidelines") | alias | Synonym: "working from home" ≠ "remote work policy" |

**Vector failures** (3/10 queries scored 0.00):

| Query | Type | Why Vector Failed |
|-------|------|------------------|
| `exact-fundraising` ("Series A fundraising") | exact | Exact keyword present in doc but embedding similarity missed it |
| `semantic-fundraising` ("raising money for startup") | semantic | BM25 caught partial keyword overlap ("fundraising") that vector missed |
| `hard-partial` ("nouns not verbs") | semantic | Partial phrase in API design doc — keyword match stronger than embedding similarity |

### What the Results Prove

**BM25 and vector search have complementary blind spots.** BM25 fails on synonyms, concepts, and topic-level queries. Vector fails on exact phrases, partial keyword matches, and some domain-specific terminology. Neither alone exceeds 0.700 precision.

**Hybrid fusion eliminates both failure modes.** By combining BM25 and vector via Reciprocal Rank Fusion, every query finds its target — perfect 1.000 across all metrics. This is the core value proposition of QMD's architecture, and `qmd bench` makes it measurable.

**The reranker doesn't differentiate on this small corpus.** Both `hybrid` (no reranking) and `full` (with LLM reranking) score 1.000. With only 6 documents, there's little noise to suppress. On a larger corpus with more false positives, the reranker would improve precision by demoting irrelevant results.

**This analysis is exactly the kind of insight that should be in the documentation.** A user running bench for the first time should understand *why* hybrid beats both individual backends — it's the strongest argument for QMD's existence.

---

## Comparison to v2.0.1 Onboarding Feedback

The [QMD Onboarding & Usage Feedback (v2.0.1)](docs/QMD-ONBOARDING-FEEDBACK-v2.0.1.md) document, written by a first-time agent user, identified systemic UX patterns in the CLI. Every pattern identified there recurs in `qmd bench`:

| v2.0.1 Pattern | Bench Equivalent |
|----------------|-----------------|
| **UX-1: No per-subcommand help** — `qmd collection add --help` prints global help | `qmd bench --help` prints global help. No argument signatures, no fixture format, no examples. |
| **UX-3: Happy path is hidden** — `qmd collection add .` is the correct command but undiscoverable | The correct setup (`collection add + embed + bench`) is a 3-step sequence documented nowhere. |
| **UX-5: Tips are the best docs but in the wrong place** — correct syntax buried in `qmd status` Tips | The best bench description is in the CHANGELOG, not README or --help. |
| **BUG-2: Silent misbehavior** — positional `context add` silently misroutes with no error | Bench silently produces all-zero results when the collection doesn't exist. No warning, no error. |
| **UX-2: Path resolution is surprising** — `src/bench/fixtures/example.json` is a source path | The error message points to a source code path that only works inside the git repo. npm users can't resolve it. |
| **Overall: "The easy path leads to failure, the hard-to-find path leads to success"** | Running bench without setup → 2 minutes of zeros. The correct path requires reading source code. |

**The pattern is consistent**: QMD builds excellent features and then under-communicates them. The search pipeline, the context system, the MCP tools, and now bench — all powerful, all poorly documented.

---

## Recommendations

### README Additions

Add a **Benchmarking** section after "Index Maintenance" (or within a new "Quality & Evaluation" section):

```markdown
### Benchmarking

Measure search quality across all backends with `qmd bench`:

\`\`\`sh
# Set up the example benchmark (one-time)
qmd collection add test/eval-docs --name eval-docs
qmd embed

# Run the benchmark
qmd bench src/bench/fixtures/example.json

# JSON output for programmatic analysis
qmd bench src/bench/fixtures/example.json --json
\`\`\`

The benchmark runs each query against 4 backends and reports precision, recall,
MRR, and F1:

| Backend | What It Tests | LLM Required |
|---------|--------------|--------------|
| `bm25` | Keyword search only | No |
| `vector` | Semantic similarity only | Embedding model |
| `hybrid` | BM25 + vector fusion (no reranking) | Embedding model |
| `full` | Full pipeline with LLM reranking | All models |

**Score interpretation:** 1.00 = perfect (found all expected documents in top
results), 0.00 = complete miss. The example fixture typically shows bm25 ~0.50,
vector ~0.70, and hybrid/full ~1.00 — demonstrating why hybrid search matters.

#### Custom Fixtures

Create your own benchmarks for your collections:

\`\`\`json
{
  "description": "My benchmark",
  "version": 1,
  "collection": "my-collection",
  "queries": [
    {
      "id": "find-auth",
      "query": "authentication flow",
      "type": "semantic",
      "description": "Should find the auth design doc",
      "expected_files": ["docs/auth-design.md"],
      "expected_in_top_k": 3
    }
  ]
}
\`\`\`

Query types: `exact` (keyword match), `semantic` (conceptual), `topical`
(theme-level), `cross-domain` (connecting distant concepts), `alias` (synonym
match). These are labels for grouping results — they don't change search
behavior.
```

### --help Improvements

Replace the current one-liner:
```
qmd bench <fixture.json>      - Run search quality benchmarks against a fixture file
```

With:
```
qmd bench <fixture.json>      - Measure search quality (P@k, recall, MRR, F1)
  qmd bench fixture.json              # Table output to terminal
  qmd bench fixture.json --json       # JSON for programmatic analysis
  qmd bench fixture.json -c my-coll   # Override fixture's collection
  See: qmd bench --example            # Print example fixture JSON
```

Consider adding `qmd bench --example` that prints the fixture JSON schema to stdout, so users don't need to find the file on disk.

### Error Messages & Guardrails

**1. Validate collection exists before running:**

Current behavior: silently runs 40 queries against the wrong scope, returns all zeros.

Proposed:
```
$ qmd bench fixture.json
Error: Collection 'eval-docs' (specified in fixture) not found.

Available collections: tweet, qmd, pipecat, ...

To set up the example benchmark:
  qmd collection add test/eval-docs --name eval-docs
  qmd embed
  qmd bench src/bench/fixtures/example.json
```

**2. Warn on all-zero results:**

If every query scores 0.00 across all backends, append:
```
⚠ All queries scored 0.00. This usually means:
  - The collection specified in the fixture is not indexed
  - The expected_files paths don't match your index
  Run 'qmd ls <collection>' to verify indexed files.
```

**3. Fix the fixture path for npm users:**

Either:
- Ship a `qmd bench --example` command that prints the example fixture to stdout
- Use `qmd bench --init` to copy the example fixture to the current directory
- Change the error message to show how to find it: `See: $(npm root -g)/@tobilu/qmd/src/bench/fixtures/example.json`

### Output Improvements

**1. Add a metric legend on first run:**

```
Metrics:
  P@k    Precision at k — fraction of top-k results that are relevant
  Recall Fraction of expected files found anywhere in results  
  MRR    Mean Reciprocal Rank — how high the first hit appears (1/rank)
  F1     Harmonic mean of precision and recall
  1.00 = perfect, 0.00 = not found
```

This could be shown by default on first run and suppressed with `--no-legend` or when using `--json`.

**2. Highlight failures in the per-query table:**

Color-code or mark zero-score rows so the user can immediately see which queries failed:
```
exact-fundraising         vector    0.00  0.00   0.00  0.00      23ms  ← MISS
```

**3. Add a one-line interpretation to the summary:**

```
Summary:
  bm25     P@k= 0.500 ...   (5/10 queries found target)
  vector   P@k= 0.700 ...   (7/10 queries found target)
  hybrid   P@k= 1.000 ...   (10/10 queries found target) ✓
  full     P@k= 1.000 ...   (10/10 queries found target) ✓
```

### Fixture Documentation

Add a `docs/BENCHMARKS.md` or a section in the README explaining:

1. **Fixture JSON schema** — all fields, required vs optional, what `expected_in_top_k` controls
2. **Query types** — what each type label means (they're for grouping, not routing)
3. **How to write expected_files** — these are collection-relative paths as they appear in `qmd ls`
4. **How to create a benchmark for your own data** — step-by-step workflow
5. **Interpreting results** — what 0.500 vs 1.000 means, when to worry, what levers to pull

---

## Scorecard

| Aspect | Rating | Notes |
|--------|--------|-------|
| **Feature quality** | 9/10 | 4 backends, 5 metrics, clean tabular output, JSON export, collection override |
| **README coverage** | 0/10 | Zero mentions in 945 lines |
| **--help coverage** | 1/10 | One line, no examples, no fixture format |
| **Error handling** | 1/10 | Silent all-zero results on missing collection, source-code path in error message |
| **Output clarity** | 5/10 | Clean table, but no metric legend, no failure highlighting, no interpretation |
| **Fixture documentation** | 1/10 | Only discoverable by reading source code |
| **Agent discoverability** | 0/10 | Not in MCP, not in skill docs, not in README |
| **Setup instructions** | 0/10 | 3-step setup sequence documented nowhere |

**Overall: The feature is ready. The communication isn't.**

---

## Appendix: Full Benchmark Output

Run on 2026-04-07, Apple M3, v2.1.0, eval-docs collection (6 documents, embedded).

```
Query                     Backend  P@k    Recall  MRR    F1     ms      
----------------------------------------------------------------------
exact-api                 bm25      1.00  1.00   1.00  1.00       9ms
exact-api                 vector    1.00  1.00   1.00  1.00     590ms
exact-api                 hybrid    1.00  1.00   1.00  1.00    3384ms
exact-api                 full      1.00  1.00   1.00  1.00    1475ms

exact-fundraising         bm25      1.00  1.00   1.00  1.00     314ms
exact-fundraising         vector    0.00  0.00   0.00  0.00      23ms
exact-fundraising         hybrid    1.00  1.00   1.00  1.00     127ms
exact-fundraising         full      1.00  1.00   1.00  1.00     920ms

exact-cap                 bm25      1.00  1.00   1.00  1.00      41ms
exact-cap                 vector    1.00  1.00   1.00  1.00      30ms
exact-cap                 hybrid    1.00  1.00   1.00  1.00      22ms
exact-cap                 full      1.00  1.00   1.00  1.00     436ms

semantic-rest             bm25      0.00  0.00   0.00  0.00     177ms
semantic-rest             vector    1.00  1.00   1.00  1.00      33ms
semantic-rest             hybrid    1.00  1.00   1.00  1.00    2116ms
semantic-rest             full      1.00  1.00   1.00  1.00     449ms

semantic-fundraising      bm25      1.00  1.00   1.00  1.00      24ms
semantic-fundraising      vector    0.00  0.00   0.00  0.00      22ms
semantic-fundraising      hybrid    1.00  1.00   1.00  1.00      28ms
semantic-fundraising      full      1.00  1.00   1.00  1.00     822ms

semantic-overfitting      bm25      0.00  0.00   0.00  0.00      66ms
semantic-overfitting      vector    1.00  1.00   1.00  1.00      26ms
semantic-overfitting      hybrid    1.00  1.00   1.00  1.00    2187ms
semantic-overfitting      full      1.00  1.00   1.00  1.00     674ms

topical-launch            bm25      0.00  0.00   0.00  0.00      60ms
topical-launch            vector    1.00  1.00   1.00  1.00      27ms
topical-launch            hybrid    1.00  1.00   1.00  1.00    2018ms
topical-launch            full      1.00  1.00   1.00  1.00     528ms

cross-domain-consistency  bm25      0.00  0.00   0.00  0.00       8ms
cross-domain-consistency  vector    1.00  1.00   1.00  1.00      40ms
cross-domain-consistency  hybrid    1.00  1.00   1.00  1.00    1803ms
cross-domain-consistency  full      1.00  1.00   1.00  1.00     836ms

alias-remote              bm25      0.00  0.00   0.00  0.00      14ms
alias-remote              vector    1.00  1.00   1.00  1.00      34ms
alias-remote              hybrid    1.00  1.00   1.00  1.00    2060ms
alias-remote              full      1.00  1.00   1.00  1.00    1013ms

hard-partial              bm25      1.00  1.00   1.00  1.00      11ms
hard-partial              vector    0.00  0.00   0.00  0.00      33ms
hard-partial              hybrid    1.00  1.00   1.00  1.00      28ms
hard-partial              full      1.00  1.00   1.00  1.00     348ms

Summary:
----------------------------------------------------------------------
  bm25     P@k= 0.500 Recall= 0.500 MRR= 0.500 F1= 0.500 Avg=72ms
  vector   P@k= 0.700 Recall= 0.700 MRR= 0.700 F1= 0.700 Avg=86ms
  hybrid   P@k= 1.000 Recall= 1.000 MRR= 1.000 F1= 1.000 Avg=1377ms
  full     P@k= 1.000 Recall= 1.000 MRR= 1.000 F1= 1.000 Avg=750ms
```

---

*Report generated during a QMD v2.1.0 testing session. All outputs are from actual runs, not synthetic examples.*
