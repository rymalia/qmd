# QMD Documentation Sprint — Canonical PR Plan

**Date**: 2026-03-17 (supersedes plan-documentation-sprint-2026-03-16.md)
**Goal**: Two upstream PRs that improve qmd's documentation and help system.
**Audience**: New users, LLM agents, and community contributors.
**Strategy**: Doc-only first (low risk, easy merge), then code change (help system).

This plan incorporates corrections from three independent validation passes conducted
on 2026-03-17, each of which tested claims against the live CLI, MCP HTTP server
(`localhost:8181`), source code (`src/mcp/server.ts`, `src/cli/qmd.ts`), and upstream
GitHub issues. All factual assertions below are empirically verified.

**Prerequisite**: The local `dev` branch is 20 commits behind `main`. Several
features referenced here (embed memory flags, launcher lockfile fix, sqlite-vec
error handling) exist in `main` but not in `dev` source. **Rebase `dev` onto `main`
before beginning any PR work.** The documentation PR targets `upstream/main`, so all
assertions must be validated against that branch.

---

## Evidence / Motivation

### From our own testing (Mar 12 stress-test sessions, re-validated Mar 17)

- MCP `get` tool supports **both** `fromLine` (explicit parameter) **and** colon
  syntax in the file path (`file.md:50`). Colon syntax is a convenience fallback used
  only when `fromLine` is not provided (`server.ts:361-368`). Neither is documented.
- `collections` is array-only in MCP (`z.array(z.string())` at `server.ts:295`).
  Singular `collection` is silently stripped by Zod — no error, no filtering. Agents
  using the wrong parameter name get unscoped results with no feedback.
- Unknown/invalid MCP parameters (e.g. `bogusParam`) are silently stripped. No error,
  no warning. This is a Zod default (no `.strict()`) — worth noting for agent authors.
- `-c` flag is a global name lookup, not pwd-relative — works from any directory.
  Nowhere documented.
- MCP server `buildInstructions` (`server.ts:105`) tells LLMs to scope with
  `collection` (singular), but the schema uses `collections` (plural array). This
  actively misleads agents into using a parameter name that gets silently ignored.

### From upstream issues (all confirmed open, no maintainer activity)

- **#25**: Error message says `qmd add .` but command is `qmd collection add .`
- **#181**: Multi-collection `-c` returns fewer results — confirmed bug, not just
  confusion. Post-filter on global top-K causes results from smaller collections to
  be pushed off the list by higher-scoring matches in larger collections.
- **#217**: Multi-collection false empty results — same root cause as #181, better
  repro. Identifies commit `640ac13` as the source.
- **#372**: `file` param should be `path` for LLM discoverability (PR #373 open,
  not merged)
- **#394**: `qmd skill install` packaging bug — zip contains multiple `.md` files,
  causing install failure

### From CHANGELOG features missing in README

- `--intent` (v1.1.5) — fully wired in CLI (`qmd.ts:2347`), works on `query` and
  `vsearch`. Absent from README Options section. Present in SYNTAX.md and in `--help`
  EBNF grammar examples, but NOT listed as a named flag in `--help` Search options.
- `--explain` (v1.1.2) — appears in README Options section and one CLI example, but
  **no sample output** showing what the score trace JSON looks like.
- `--no-rerank` (v1.1.2) — in `--help` only. Not in README anywhere.
- `-C`/`--candidate-limit` (v1.1.2) — in `--help` only. Not in README anywhere.
- `collection include/exclude` (v1.1.0) — not in README. Subcommands exist and work.
- `collection update-cmd` (v1.1.0) — not in README. Subcommand exists and works.
- `collection show` (v1.1.0) — not in README. Subcommand exists and works.
- Multiple `-c` flags (v1.0.7) — not in README. Works with OR semantics across
  named collections.
- `qmd://` URI scheme (v0.5.0) — used 8 times in README as bare examples, never
  formally defined as a scheme with semantics (inheritance, collection mapping).
- `vector-search` and `deep-search` aliases (undocumented) — exist in source
  (`qmd.ts:2958, 2971`) as aliases for `vsearch` and `query` respectively.
- `--max-docs-per-batch` and `--max-batch-mb` embed flags — exist in
  `upstream/main:src/cli/qmd.ts` (commit `809aa36` via PR #370). Control memory
  usage during `qmd embed`. Present in compiled dist and `--help` but absent from
  README.

### SYNTAX.md has a broken MCP example

`docs/SYNTAX.md` line 136 shows a `q` string parameter:
```json
{ "q": "lex: CAP theorem\nvec: consistency vs availability", ... }
```
This parameter does not exist. Both the MCP `query` tool and the REST `/query`
endpoint only accept `searches` (array). Testing against the live server returns
`400: "Missing required field: searches (array)"`. The `q` format was likely a
pre-v2.0 interface. The `collections` field in the same example is already correct
(plural array) — no fix needed there.

### Help system analysis

- All `--help` invocations produce the **exact same 85-line output** (MD5:
  `6baa1a6854e5f180b7e2d64dc74541de`). This includes `qmd query --help`,
  `qmd get --help`, `qmd collection --help`, `qmd embed --help`, `qmd mcp --help`,
  etc. — all identical.
- No per-subcommand help exists via `--help`. However, bare `qmd collection` (no
  args) already shows a subcommand directory listing all 8 subcommands (add, list,
  remove, rename, show, update-cmd, include, exclude). Bare `qmd query` (no args)
  shows an error, not help.
- EBNF grammar shown for every command including `embed` and `mcp` where it's
  irrelevant.
- `--from` flag (for `qmd get`) is in the README and works in the CLI, but does NOT
  appear in `--help` output.
- Modern CLIs (uv, cargo, gh, rg) all use tiered/progressive disclosure.

---

## Phase 1: Documentation PR (README.md + docs/SYNTAX.md)

**Title**: `docs: improve CLI reference, collection flag semantics, and MCP parameter documentation`

No code changes. Pure markdown. Should be easy for tobi to review and merge.

### README.md changes

#### 1. Add `-c`/`--collection` semantics section

After the existing "Options" section, add a subsection:

```markdown
### Collection Filtering

The `-c`/`--collection` flag filters results by collection **name** (as shown by
`qmd collection list`). Collections are a global registry — you can search any
collection from any directory:

    qmd search "auth" -c notes          # single collection
    qmd search "auth" -c notes -c docs  # multiple collections (OR)

When no `-c` flag is given, all default-included collections are searched.
Collections marked as excluded (`qmd collection exclude <name>`) are skipped
unless explicitly named with `-c`.

**Note**: When using multiple `-c` flags, results are drawn from a global top-K
pool and then filtered. If one collection dominates the rankings, results from
smaller collections may not appear at the default limit. Use `-n` or `--all` to
see more.
```

#### 2. Document missing subcommands

Add to the "Collection Management" section:

```sh
# Show collection details (path, pattern, include status, context count)
qmd collection show myproject

# Include or exclude a collection from default queries
qmd collection include myproject
qmd collection exclude myproject

# Set a command to run before every `qmd update` (e.g., git pull)
qmd collection update-cmd myproject 'git pull --rebase'
# Clear the update command
qmd collection update-cmd myproject
```

#### 3. Document missing search options

Add to the "Options" section:

```sh
# Search refinement
--intent "context"         # Disambiguate query (e.g., --intent "web performance")
--no-rerank                # Skip LLM reranking (use RRF scores only, faster on CPU)
-C, --candidate-limit <n>  # Max candidates to rerank (default 40, lower = faster)
```

For `--explain`, add a brief example of what the output looks like (sample score
trace). The flag is already listed in the Options section at line 625 and shown in
a CLI example at line 689, but users have no idea what the output looks like. Add
an example output block after the existing CLI example.

For `--intent`, add an example to the Examples section:

```sh
# Disambiguate "performance" — find web perf, not sports
qmd query --intent "web page load times" "performance"
```

#### 4. Document embed memory flags

Add to the "Generate Vector Embeddings" section or "Options":

```sh
# Embedding memory control (useful for memory-constrained systems)
--max-docs-per-batch <n>   # Limit docs per embedding batch
--max-batch-mb <n>         # Limit batch size in MB
```

#### 5. Add search command aliases

Add to the "Search Commands" table:

```
┌──────────────────────────────────────────────────────────────────┐
│                        Search Modes                              │
├──────────────┬───────────────────────────────────────────────────┤
│ search       │ BM25 full-text search only                       │
│ vsearch      │ Vector semantic search only                      │
│ query        │ Hybrid: FTS + Vector + Query Expansion + Rerank  │
├──────────────┼───────────────────────────────────────────────────┤
│ vector-search│ Alias for vsearch                                │
│ deep-search  │ Alias for query                                  │
└──────────────┴───────────────────────────────────────────────────┘
```

#### 6. Add MCP parameter reference

After the existing MCP section, add a compact table:

```markdown
#### MCP Tool Parameters

| Tool | Parameter | Type | Notes |
|------|-----------|------|-------|
| `query` | `searches` | array | Typed sub-queries (lex/vec/hyde). Required. First gets 2x weight. |
| `query` | `collections` | string[] | Filter by collection names (OR). Array only — singular `collection` is silently ignored. |
| `query` | `intent` | string | Disambiguation context (does not search on its own) |
| `query` | `limit` | number | Max results (default 10) |
| `query` | `minScore` | number | Minimum relevance 0-1 (default 0) |
| `query` | `candidateLimit` | number | Max candidates to rerank (default 40) |
| `get` | `file` | string | File path, docid (`#abc123`), or `path:line` for offset |
| `get` | `fromLine` | number | Start from this line (1-indexed). Alternative to `path:line` colon syntax. |
| `get` | `maxLines` | number | Limit returned lines |
| `get` | `lineNumbers` | boolean | Include line numbers (default false) |
| `multi_get` | `pattern` | string | Glob pattern or comma-separated list |
| `multi_get` | `maxBytes` | number | Skip files larger than N (default 10240) |
| `multi_get` | `maxLines` | number | Limit lines per file |
| `multi_get` | `lineNumbers` | boolean | Include line numbers (default false) |

Unknown parameters are silently ignored (not rejected). Double-check parameter
names if results seem unscoped.
```

### docs/SYNTAX.md changes

#### 1. Fix MCP/HTTP API section — remove broken `q` parameter

The first JSON example in the MCP/HTTP API section uses a `q` string parameter that
neither the MCP tool nor the REST endpoint accepts. Replace the entire first example
block with the `searches` array format:

**Current (broken)**:
```json
{
  "q": "lex: CAP theorem\nvec: consistency vs availability",
  "collections": ["docs"],
  "limit": 10
}
```

**Replace with**:
```json
{
  "searches": [
    { "type": "lex", "query": "CAP theorem" },
    { "type": "vec", "query": "consistency vs availability" }
  ],
  "collections": ["docs"],
  "limit": 10
}
```

Note: The `collections` field is already correct (plural array). No change needed
there — this was incorrectly flagged in the original plan.

#### 2. Add `-c` CLI flag reference

SYNTAX.md covers query syntax in detail but never mentions collection scoping.
Add a "Scoping" section after "Constraints":

```markdown
## Scoping

Restrict queries to specific collections with `-c` (CLI) or `collections` (MCP/SDK):

    # CLI
    qmd query -c docs "how does auth work"
    qmd query -c docs -c notes $'lex: auth\nvec: authentication flow'

    # MCP / HTTP
    { "searches": [...], "collections": ["docs", "notes"] }

Collections are identified by name (see `qmd collection list`). Without scoping,
all default-included collections are searched. Collections marked as excluded
(`qmd collection exclude <name>`) are skipped unless explicitly named.
```

---

## Phase 2: Progressive Help PR (src/cli/qmd.ts)

**Title**: `feat(cli): per-subcommand help with contextual options and examples`

Code change to the help system. Larger, but well-scoped.

### Design principles

1. **Top-level help is a command directory** — commands + 1-line descriptions + global options
2. **Per-command help shows only relevant content** — that command's flags, usage, and 1-2 examples
3. **The EBNF grammar lives in `query --help` only** — not `get --help` or `embed --help`
4. **No content is lost** — everything currently shown is still accessible somewhere
5. **Keep it maintainable** — help text close to the command handler, not in a separate file

### Implementation sketch

The current help is a single function that emits 85 identical lines for every
subcommand. Replace with:

```typescript
// Top-level: compact directory
function showHelp() { ... }

// Per-command: contextual
function showQueryHelp() { ... }     // includes EBNF grammar, search options, --intent
function showSearchHelp() { ... }    // search options only (subset), no grammar
function showGetHelp() { ... }       // file/docid syntax, --from, line slicing
function showMultiGetHelp() { ... }  // glob patterns, maxBytes
function showCollectionHelp() { ... } // subcommands: add/list/remove/show/include/exclude
function showContextHelp() { ... }   // add/list/rm with qmd:// examples
function showEmbedHelp() { ... }     // embed-specific flags
function showMcpHelp() { ... }       // transport modes, daemon lifecycle
```

Note: Bare `qmd collection` (no args) already shows a subcommand directory listing
all 8 subcommands. The Phase 2 work should follow this pattern for other commands
and make `--help` per-command rather than replacing existing bare-command behavior.

### Approximate line counts per help screen

| Command | Current | Target |
|---------|---------|--------|
| `qmd --help` (top) | 85 | ~25 (commands + global opts) |
| `qmd query --help` | 85 (same) | ~35 (grammar + search opts + examples) |
| `qmd search --help` | 85 (same) | ~15 (search opts + examples) |
| `qmd get --help` | 85 (same) | ~12 (file/docid syntax + --from + line slicing) |
| `qmd collection --help` | 85 (same) | ~15 (subcommands + examples) |
| `qmd embed --help` | 85 (same) | ~10 (embed flags) |

### `--help` gaps to fix in this phase

These flags work in the CLI but do not appear in the `--help` Search options block:

- `--intent` — only present as `intent:` in the EBNF grammar examples
- `--from` — shown in `qmd get` usage error message but absent from `--help`

### Risks and mitigations

- **Risk**: Help text drifts from actual flags → **Mitigation**: add a test that
  parses each help function's output and checks mentioned flags exist in the arg parser
- **Risk**: Community PRs update one help function but not another → **Mitigation**:
  keep per-command help close to the command handler (same function or adjacent),
  not in a separate help module
- **Risk**: Tobi may prefer the current flat style → **Mitigation**: file an issue
  first to gauge interest before writing the PR

---

## Bonus: Small code fixes to consider bundling

These are tiny fixes found during validation. Each is one line. They could ride
along with Phase 1 (making it "docs + trivial fix") or be filed separately.

### Fix `buildInstructions` parameter name (server.ts:105)

Current: `Collections (scope with \`collection\` parameter):`
Should be: `Collections (scope with \`collections\` parameter):`

This actively misleads LLMs into using a singular parameter name that gets silently
stripped by Zod, resulting in unscoped searches. One-word fix.

---

## Upstream PR strategy

### For Phase 1 (docs)
- File PR directly — doc improvements are low-friction to review
- Reference issues #25, #181, #217, #372 in the PR body as motivation
- Keep the diff focused: README.md + docs/SYNTAX.md only
- The `buildInstructions` fix is a one-word change in `server.ts` — consider
  including it or filing separately

### For Phase 2 (progressive help)
- **File an issue first** proposing the help restructure, linking to Phase 1 PR
- Reference the clig.dev guidelines and the uv/cargo/gh precedent
- Wait for signal from tobi before writing the code
- If the issue gets positive response, then submit the PR

### What NOT to include in these PRs
- Bug fixes (hyphen handling, multi-collection filtering) — those are separate PRs
- SDK documentation — the README SDK section is already comprehensive
- Architecture diagrams — already well-done
- New features or behavioral changes — these are purely additive documentation

---

## Validation trail

This plan was validated by three independent reviews on 2026-03-17:

1. `docs/audit-01-2026-03-17-documentation-sprint-validation.md` — confirmed core
   assumptions but missed all four critical errors. Contributed unique findings:
   undocumented search aliases.
2. `docs/validation-02-of-documentation-sprint-2026-03-17.md` — caught all four
   critical errors. Contributed unique findings: bare `qmd collection` help,
   `--from` flag gap, `lineNumbers` on `multi_get`.
3. `docs/documentation-sprint-plan—empirical-validation-report-2026-03-17.md` —
   caught all four critical errors plus three new issues: `buildInstructions`
   singular/plural mismatch, `multi_get` isError inconsistency, `bun.lock`
   reintroduction.

Discarded claims from reviews:
- Audit 01's recommendation to "proceed with implementation" without corrections —
  would have produced a PR with factually incorrect assertions.
- Original plan's claim that `fromLine` doesn't exist — it does (`server.ts:355`).
- Original plan's claim that SYNTAX.md uses singular `collection` — it already uses
  plural `collections`.
