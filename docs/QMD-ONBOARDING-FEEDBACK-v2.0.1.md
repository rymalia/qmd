---
title: QMD Onboarding & Usage Feedback
date: 2026-03-31
type: feedback
tags: [qmd, onboarding, mcp, cli, ux, search, bug-report]
project: qmd
version: v2.0.1
related: [minutes, smaug, kuato]
---

# QMD Onboarding & Usage Feedback

> Feedback from a first-time QMD user (Claude Code agent) onboarding the `minutes` project into a workspace with 39 existing QMD collections. Session date: 2026-03-31.

## Executive Summary

QMD's **query capabilities are excellent** — hybrid lex+vec+hyde search, collection scoping, context descriptions, cross-project discovery. The search API (both MCP and CLI) is well-documented and powerful. The **collection and context management CLI is where the friction lives**: missing argument signatures, silent misbehavior on ambiguous input, and the correct syntax hidden in `qmd status` Tips rather than `--help`.

**Overall rating**: 8/10 for searching, 4/10 for collection/context management CLI.

---

## Table of Contents

- [Test Environment](#test-environment)
- [Onboarding Walkthrough](#onboarding-walkthrough)
  - [Step 1: Discovery — What is QMD?](#step-1-discovery--what-is-qmd)
  - [Step 2: Adding a Collection](#step-2-adding-a-collection)
  - [Step 3: Adding Context Descriptions](#step-3-adding-context-descriptions)
  - [Step 4: Querying](#step-4-querying)
- [Use Case Testing](#use-case-testing)
  - [Use Case 1: Cross-Project Pattern Discovery](#use-case-1-cross-project-pattern-discovery)
  - [Use Case 2: Architectural Decision Archaeology](#use-case-2-architectural-decision-archaeology)
  - [Use Case 3: Prior Solution Lookup](#use-case-3-prior-solution-lookup)
  - [Use Case 4: Scoped Multi-Collection Queries](#use-case-4-scoped-multi-collection-queries)
- [Bug Reports](#bug-reports)
  - [BUG-1: Hyphen-as-Negation in vec/hyde Queries](#bug-1-hyphen-as-negation-in-vechyde-queries)
  - [BUG-2: Positional context add Silently Misroutes](#bug-2-positional-context-add-silently-misroutes)
- [UX Issues](#ux-issues)
  - [UX-1: No Per-Subcommand Help](#ux-1-no-per-subcommand-help)
  - [UX-2: collection add Path Resolution is Surprising](#ux-2-collection-add-path-resolution-is-surprising)
  - [UX-3: The Happy Path is Hidden](#ux-3-the-happy-path-is-hidden)
  - [UX-4: Silent Context Overwrite](#ux-4-silent-context-overwrite)
  - [UX-5: qmd status Tips are the Best Docs but in the Wrong Place](#ux-5-qmd-status-tips-are-the-best-docs-but-in-the-wrong-place)
  - [UX-6: Reranking Latency](#ux-6-reranking-latency)
- [What Exceeded Expectations](#what-exceeded-expectations)
- [Suggested Help Text Improvements](#suggested-help-text-improvements)
- [Summary Scorecard](#summary-scorecard)

---

## Test Environment

- **QMD index**: 2854 docs across 39 collections (pre-test), 2895 after adding minutes
- **New collection added**: `minutes` (41 markdown files, 165 embedded chunks)
- **Hardware**: Apple M3, 11.8 GB VRAM
- **Models**: embeddinggemma (embedding), Qwen3-Reranker-0.6B (reranking), qmd-query-expansion-1.7B (generation)
- **Interfaces tested**: QMD MCP tools (via Claude Code) and QMD CLI

---

## Onboarding Walkthrough

### Step 1: Discovery — What is QMD?

**First action**: Called `mcp__qmd__status` to understand the system.

**Result**: Excellent. The status response immediately showed all 39 collections with paths, doc counts, patterns, and last-updated timestamps. It was instantly clear that `minutes` was not yet indexed. The MCP tool descriptions in the system prompt were also clear about the four tools available (status, query, get, multi_get).

**What worked well**: The MCP tool documentation for `query` is genuinely good — the strategy table (lex for exact terms, vec for concepts, lex+vec for best recall, lex+vec+hyde for complex/nuanced) is exactly what an agent needs to choose the right approach.

### Step 2: Adding a Collection

This is where friction began. The MCP tools are read-only (query, get, status) — collection management requires the CLI.

#### Attempt 1: Explicit three-arg form
```bash
qmd collection add minutes /Users/rymalia/projects/minutes '**/*.md'
```
**Result**: Created collection with path `/Users/rymalia/projects/minutes/minutes` (appended collection name as subdirectory). Zero files found.

#### Attempt 2: Dot for current directory
```bash
qmd collection add minutes . '**/*.md'
```
**Result**: Same — resolved to `.../minutes/minutes`. Zero files found.

#### Attempt 3: Shell-expanded $PWD
```bash
qmd collection add minutes "$PWD" '**/*.md'
```
**Result**: Same broken path. Zero files found.

#### Attempt 4: Trailing slash
```bash
qmd collection add minutes /Users/rymalia/projects/minutes/ '**/*.md'
```
**Result**: Same. Zero files found.

#### Attempt 5: From parent directory (finally worked)
```bash
cd /Users/rymalia/projects && qmd collection add minutes minutes '**/*.md'
```
**Result**: Path resolved to `/Users/rymalia/projects/minutes`. **41 files indexed.** This worked because the second arg `minutes` was resolved relative to the parent CWD.

#### The actual happy path (discovered later, told by user)
```bash
cd /Users/rymalia/projects/minutes && qmd collection add .
```
**Result**: Auto-derives collection name from directory basename (`minutes`), sets path to CWD, defaults pattern to `**/*.md`. **This is the correct one-command workflow.** It was not discoverable from `--help`.

**Key insight**: The single-arg `qmd collection add .` form is elegant and correct, but the help text shows `qmd collection add/list/remove/rename/show - Manage indexed folders` with zero argument signatures or examples. An agent (or new user) will guess at the multi-arg form first and hit the path resolution bug.

### Step 3: Adding Context Descriptions

Context descriptions attach human-written summaries to collections or subpaths within them. They serve dual purposes: (1) help the reranker understand what a collection contains, and (2) appear in search results alongside hits, giving the querying agent disambiguation context.

#### First attempt: Positional syntax
```bash
qmd context add minutes / "Open-source privacy-first conversation memory layer..."
qmd context add minutes docs "Development documentation: session summaries..."
```
**Result**: Both reported success (`Added context for: qmd://minutes/minutes`), but only one context was retained — the second silently overwrote the first. The path argument was being concatenated with the collection name to produce `qmd://minutes/minutes` for both, rather than creating distinct entries at `/` and `/docs`.

#### Discovery: URI syntax in `qmd status` Tips
The `qmd status` output includes a Tips section at the very bottom:
```
qmd context add qmd://<name>/ "What this collection contains"
qmd context add qmd://<name>/meeting-notes "Weekly team meeting notes"
```
This reveals the **URI-based syntax** which correctly creates separate context entries.

#### Working approach: URI syntax
```bash
qmd context add "qmd://minutes/" "Open-source privacy-first..."
qmd context add "qmd://minutes/docs" "Development documentation..."
qmd context add "qmd://minutes/.claude/plugins/minutes" "Claude Code plugin: 12 skills..."
```
**Result**: `Contexts: 3` confirmed. Each context at a distinct path. The root context showed `(collection root)` annotation — a clear signal it resolved correctly.

**Comparison with existing collections**: Other collections like `kuato` (5 contexts), `qmd` (4 contexts), and `smaug` (3 contexts) all use this URI scheme successfully, visible in `qmd status` output.

### Step 4: Querying

After indexing + embedding + adding contexts, queries immediately returned relevant results from the new collection. The embedding step took 13s for 41 docs / 165 chunks on an M3.

---

## Use Case Testing

### Use Case 1: Cross-Project Pattern Discovery

**Question**: Minutes deals with streaming whisper transcription. What related knowledge exists across sibling projects?

**MCP query**:
```json
{
  "searches": [
    {"type": "vec", "query": "streaming audio transcription with low latency for voice assistants in realtime"},
    {"type": "lex", "query": "streaming transcription whisper latency"}
  ],
  "limit": 8
}
```

**Results** (8 hits across 6 projects):
| Score | Project | Document | Why it's relevant |
|-------|---------|----------|-------------------|
| 93% | smaug | `knowledge/tools/whisper-flow.md` | Bookmarked tool for real-time whisper transcription |
| 56% | projects-root-folder | `asr-speech-to-text-projects-todo.md` | ASR project tracking with streaming engines |
| 47% | fluid-audio | `sources/fluidaudiocli/readme.md` | CLI for audio transcription |
| 46% | smaug | `knowledge/tools/claude-stt.md` | Claude STT with live streaming dictation |
| 46% | smaug | `readme.md` | Bookmarks mentioning whisper-flow |
| 45% | soniqo-speech-swift | `docs/audio/playback.md` | Streaming audio playback architecture |
| 45% | pipecat-docs | `guides/learn/speech-to-text.mdx` | STT pipeline placement docs |
| 45% | mac-whisper-speedtest | `docs/feature-plan-moonshine-implementation.md` | Moonshine streaming mode plan |

**Verdict**: **Exceeded expectations.** QMD surfaced a bookmarked tweet about `whisper-flow` from smaug's knowledge archive — I never would have searched a bookmark archiver for transcription tools. This is the cross-project discovery use case working exactly as intended.

### Use Case 2: Architectural Decision Archaeology

**Question**: Why use markdown as source of truth instead of a database?

**MCP query**:
```json
{
  "searches": [
    {"type": "vec", "query": "why use markdown files as the source of truth instead of a database for storing structured data"},
    {"type": "hyde", "query": "We chose markdown with YAML frontmatter as the canonical storage format instead of SQLite because it avoids vendor lockin, works with any tool that reads text files, and enables grep workflows. The database is a derived cache that can be rebuilt from the markdown files at any time."}
  ],
  "limit": 6
}
```

**Results** (top hits):
| Score | Project | Document | Why it's relevant |
|-------|---------|----------|-------------------|
| 88% | tweet | `docs/feature-ideas.md` | Ecosystem reference as single source of truth |
| 50% | kuato | `claude.md` | File-based vs. PostgreSQL backend design |
| 47% | smaug | `knowledge/articles/files-are-all-you-need.md` | Article arguing filesystems are sufficient for AI agents |
| 43% | smaug | `bookmarks.md` | Tweet: "A well-organized filesystem with semantic search might be all you need" |
| 38% | dewey-corpus-pipeline | `readme.md` | "The enriched tweet data does not live in SQL..." |

**Verdict**: **Exceeded expectations.** The `hyde` query (writing a hypothetical answer) found the "Files Are All You Need" article in smaug — a bookmarked tweet arguing that filesystems beat databases for agent use cases. This is exactly the philosophical backing for minutes' markdown-as-database design, surfaced from a completely different project.

### Use Case 3: Prior Solution Lookup

**Question**: How to handle macOS TCC permissions in a desktop app?

**CLI query**:
```bash
qmd query $'vec: how to handle macOS permissions and TCC prompts in a desktop app\nlex: TCC permissions accessibility "input monitoring"' -n 6
```

**Results**:
| Score | Project | Document | Why it's relevant |
|-------|---------|----------|-------------------|
| 93% | minutes | `docs/desktop-development.md` | TCC permission split identity strategy |
| 56% | minutes | `docs/debug-dictation-hotkey.md` | CGEventTap + Input Monitoring debugging |
| 42% | minutes | `claude.md` | TCC-sensitive build workflow |
| 42% | minutes | `readme.md` | install-dev-app.sh permission setup |
| 34% | soniqo-speech-swift | `readme-zh.md` | Apple Silicon ML models (tangential) |
| 33% | minutes | `plan.md` | Permission policy design |

**Verdict**: **Met expectations.** The newly indexed minutes collection dominated results, with its context descriptions visibly boosting relevance. The 93% top hit contained exactly the TCC strategy (split bundle identity for prod vs. dev). The context I added earlier appeared in the output alongside each hit.

### Use Case 4: Scoped Multi-Collection Queries

**Question**: Can you limit search to specific collections to avoid noise from the 39-collection workspace?

**MCP query** (scoped to `minutes` + `smaug` only):
```json
{
  "searches": [
    {"type": "lex", "query": "whisper transcription streaming"},
    {"type": "vec", "query": "approaches to transcribing audio with whisper in realtime"}
  ],
  "collections": ["minutes", "smaug"],
  "limit": 6
}
```

**CLI equivalent**:
```bash
qmd query $'lex: whisper transcription streaming\nvec: approaches to transcribing audio with whisper in realtime' -c minutes -c smaug -n 6
```

**Results**: All 6 hits from minutes or smaug only. Zero leakage from pipecat (35 docs), pipecat-docs (328 docs), fluid-audio (37 docs), mac-whisper-speedtest (65 docs), or any other audio-related collection.

**Why this matters**: Without scoping, a query about "whisper transcription" in a workspace with this many audio-related projects would return a wall of tangentially relevant hits. Scoping to the two projects you're actively working across gives focused, actionable results.

**Verdict**: **Met expectations.** Both MCP (`"collections": [...]`) and CLI (`-c name -c name`) syntax work correctly. Collection scoping is an essential feature for large workspaces.

---

## Bug Reports

### BUG-1: Hyphen-as-Negation in vec/hyde Queries

**Status**: Known bug (already filed).

**Severity**: High — hyphens are pervasive in technical writing.

**Description**: Hyphens in `vec` and `hyde` query text are parsed as BM25 negation operators, causing query rejection. The `lex` query type legitimately supports `-term` negation, but `vec` and `hyde` queries are natural language — hyphens should be treated as literal characters.

**Reproduction**:
```json
{"type": "vec", "query": "real-time streaming transcription"}
```
Error: `Negation (-term) is not supported in vec/hyde queries. Use lex for exclusions.`

**Affected terms encountered during this session**:
- `real-time` (extremely common)
- `lock-in`
- `grep-based`
- `push-to-talk`
- `text-to-speech`
- `cross-platform`
- `anti-hallucination`
- Any kebab-case identifier (CSS class names, CLI flags, package names, URL slugs)

**Workaround**: Rewrite compound words without hyphens ("realtime", "lockin") or rephrase entirely. This is error-prone and produces less natural queries.

**Suggested fixes** (in priority order):
1. **Don't parse negation in vec/hyde queries at all** — these query types are natural language, not BM25 syntax. The negation operator has no meaning in a vector similarity search.
2. **If parsing must happen, require whitespace before `-`** — `"real-time"` is clearly not negation, while `foo -bar` is.
3. **At minimum, improve the error message**: Instead of just "Negation (-term) is not supported", suggest: "Tip: hyphens in compound words like 'real-time' are parsed as negation. Try 'realtime' or rephrase."

**Impact**: This bug was hit 4 times during a single onboarding session. Every time, it required rewriting the query and losing the natural phrasing. For an AI agent composing queries programmatically, this is particularly painful because the agent has to learn to avoid hyphens in its query generation.

### BUG-2: Positional context add Silently Misroutes

**Description**: `qmd context add <collection> <path> "text"` produces a different internal key than `qmd context add "qmd://<collection>/<path>" "text"`. The positional form concatenates the collection name into the path, causing:
1. Root contexts to land at `/minutes` instead of `/`
2. Multiple contexts to overwrite each other (same internal key)
3. No error or warning — the command reports success

**Reproduction**:
```bash
# These produce the SAME internal key (qmd://minutes/minutes):
qmd context add minutes / "Root description"
qmd context add minutes docs "Docs description"

# Only the second one survives. No warning printed.
qmd collection show minutes
# → Contexts: 1  (should be 2)
```

**Expected behavior**: Either:
- The positional form should correctly resolve paths (e.g., `minutes /` → `qmd://minutes/`, `minutes docs` → `qmd://minutes/docs`), OR
- The positional form should be deprecated with a warning pointing to the URI syntax

**Working syntax**:
```bash
qmd context add "qmd://minutes/" "Root description"        # → qmd://minutes/ (collection root)
qmd context add "qmd://minutes/docs" "Docs description"    # → qmd://minutes/docs
qmd context add "qmd://minutes/.claude" "Plugin description"  # → qmd://minutes/.claude
# All three coexist correctly. Contexts: 3
```

---

## UX Issues

### UX-1: No Per-Subcommand Help

**Current state**: `qmd collection add --help` and `qmd context add --help` both print the full global help page. There are no argument signatures, usage lines, or examples for any subcommand.

**Impact**: The user must guess argument order and semantics. For `collection add`, the argument order is `[name] [path] [pattern]` with defaults, but this isn't documented anywhere in `--help`.

**Suggested fix**: Add usage lines to each subcommand:
```
qmd collection add [name] [path] [pattern]   Index a folder (defaults: name from dirname, cwd, **/*.md)
  Examples:
    qmd collection add .                        # Current dir, auto-name, **/*.md
    qmd collection add my-notes ~/notes '*.md'  # Custom name, path, pattern
```

### UX-2: collection add Path Resolution is Surprising

**Current state**: When using the multi-arg form `qmd collection add <name> <path> <pattern>`, the path argument appears to be resolved relative to something that includes the collection name. This caused the path `/Users/rymalia/projects/minutes` to become `/Users/rymalia/projects/minutes/minutes`.

**Steps to reproduce**:
```bash
cd /Users/rymalia/projects/minutes
qmd collection add minutes /Users/rymalia/projects/minutes '**/*.md'
# Creates: /Users/rymalia/projects/minutes/minutes (WRONG)

qmd collection add minutes . '**/*.md'
# Creates: /Users/rymalia/projects/minutes/minutes (WRONG)

qmd collection add minutes "$PWD" '**/*.md'
# Creates: /Users/rymalia/projects/minutes/minutes (WRONG)
```

**Working approach**:
```bash
cd /Users/rymalia/projects
qmd collection add minutes minutes '**/*.md'
# Creates: /Users/rymalia/projects/minutes (CORRECT)
```

**Or the single-arg form** (best):
```bash
cd /Users/rymalia/projects/minutes
qmd collection add .
# Creates: /Users/rymalia/projects/minutes (CORRECT, name auto-derived as "minutes")
```

### UX-3: The Happy Path is Hidden

The simplest correct command (`qmd collection add .`) is not shown in `--help`, not in any visible documentation, and is only discoverable by:
1. Already knowing it exists, or
2. Being told by someone who knows

The multi-arg form that a new user naturally tries first is the one that breaks. This is an inverted "pit of success" — the easy path leads to failure, the hard-to-find path leads to success.

**Suggested fix**: Make `qmd collection add .` the first example in `--help`, with a comment like "# Recommended: index current directory".

### UX-4: Silent Context Overwrite

**Current state**: Adding a context that resolves to the same internal key as an existing context silently replaces it. No warning is printed.

**Expected behavior**: Print `Warning: replacing existing context at qmd://minutes/...` so the user knows they're overwriting, not appending. Alternatively, if the intent is to support one context per path, error with "Context already exists at qmd://minutes/. Use --force to replace."

### UX-5: qmd status Tips are the Best Docs but in the Wrong Place

The `qmd status` output includes a Tips section at the very bottom with correct syntax examples:
```
Tips
  Add context to collections for better search results: playwright-web-scraping, mlx-lm +14 more
    qmd context add qmd://<name>/ "What this collection contains"
    qmd context add qmd://<name>/meeting-notes "Weekly team meeting notes"
```

This is **the only place** showing the URI syntax for context add. It's also the only place showing the `collection update-cmd` feature. These tips are excellent — they just need to also appear in `--help` and any documentation.

### UX-6: Reranking Latency

**Observed**: CLI queries with reranking took 11-18 seconds for 20-33 chunks on an M3.

**Context**: This is the Qwen3-Reranker-0.6B model running on GPU. For interactive "quick check" workflows, this is noticeable. The MCP queries felt faster, possibly due to different candidate limits or caching.

**Not necessarily a bug** — reranking quality is excellent and the latency may be acceptable for the accuracy tradeoff. But worth noting for the team to consider:
- Could the default `candidateLimit` be lower for scoped queries (fewer collections = fewer candidates)?
- Is there a `--no-rerank` fast path for quick lookups? (The help mentions `--no-rerank` — this could be promoted more for interactive use.)

---

## What Exceeded Expectations

1. **Cross-project discovery is the killer feature.** Finding `whisper-flow` in smaug's bookmark archive and "Files Are All You Need" via hyde — these are connections no keyword search would make. The combination of BM25 + vector + hypothetical document embedding is genuinely powerful.

2. **Context descriptions immediately improve ranking.** After adding contexts to the minutes collection, its documents consistently scored higher than contextless collections. The context text appears in search results, giving the querying agent disambiguation signal. This creates a virtuous cycle: better contexts → better rankings → more useful results → motivation to add more contexts.

3. **The MCP query tool documentation is excellent.** The strategy table, the grammar specification, the examples — everything an agent needs to compose effective queries. This is the gold standard for MCP tool documentation.

4. **Collection scoping works cleanly.** Both MCP (`"collections": [...]`) and CLI (`-c name -c name`) correctly filter results. Essential for large workspaces where unscoped queries would return noise from 39 collections.

5. **`qmd status` output is comprehensive.** Collection list with paths, doc counts, context previews, model info, device info, and actionable tips. This single command gives complete situational awareness.

6. **hyde queries are powerful for architectural questions.** Writing a hypothetical answer paragraph surfaces philosophically aligned documents that keyword search would miss. The "Files Are All You Need" hit (an article arguing filesystems > databases for agents) was directly relevant to minutes' markdown-as-database design, found via hyde from a bookmarked tweet in a completely different project.

---

## Suggested Help Text Improvements

### Current `--help` (collection/context section):
```
Collections & context:
  qmd collection add/list/remove/rename/show   - Manage indexed folders
  qmd context add/list/rm                      - Attach human-written summaries
  qmd ls [collection[/path]]                   - Inspect indexed files
```

### Proposed `--help` (collection/context section):
```
Collections & context:
  qmd collection add [name] [path] [pattern]   - Index a folder (default: cwd, **/*.md)
    qmd collection add .                          # Index current dir, auto-name from folder
    qmd collection add my-notes ~/notes '*.md'    # Custom name, path, and pattern
  qmd collection list                           - List all collections
  qmd collection show <name>                    - Show collection details and contexts
  qmd collection remove <name>                  - Remove a collection and its documents
  qmd collection rename <old> <new>             - Rename a collection

  qmd context add <qmd://name/path> "text"     - Attach human-written summary
    qmd context add qmd://my-notes/ "Project notes and meeting logs"
    qmd context add qmd://my-notes/docs "API documentation and guides"
  qmd context list [collection]                 - List all contexts
  qmd context rm <qmd://name/path>             - Remove a context

  qmd ls [collection[/path]]                   - Inspect indexed files
```

This adds:
- Argument signatures for each subcommand
- Inline examples showing the recommended usage
- The URI syntax for context (currently only in `qmd status` Tips)
- Separate lines for each subcommand (not grouped as `add/list/remove/rename/show`)

---

## Summary Scorecard

| Area | Rating | Notes |
|------|--------|-------|
| **MCP query API** | 9/10 | Excellent documentation, powerful lex+vec+hyde combo, collection scoping |
| **MCP status tool** | 9/10 | Complete situational awareness in one call |
| **MCP get/multi_get** | 8/10 | Solid, line-range support is useful |
| **CLI query** | 8/10 | Rich output, same power as MCP, but reranking latency (11-18s) is noticeable |
| **CLI collection add** | 4/10 | Path resolution bug, no help, happy path hidden |
| **CLI context add** | 4/10 | Two syntaxes (positional vs URI), silent overwrite, URI syntax undocumented in help |
| **CLI --help** | 3/10 | No per-subcommand help, no argument signatures, no examples |
| **qmd status Tips** | 9/10 | Great examples, actionable suggestions — should be promoted to --help |
| **Cross-project discovery** | 10/10 | This is the product's standout feature |
| **Context description system** | 9/10 | Powerful once you know the URI syntax |
| **Hyphen handling** | 2/10 | Known bug, blocks natural technical queries, no helpful error message |

---

*Feedback generated during a codebase deep-dive session on the `minutes` project. All queries and results shown are from actual test runs, not synthetic examples.*
