---
title: "QMD Documentation Sprint — Canonical Plan (Refreshed)"
date: 2026-06-06
type: project
supersedes: docs/plan-documentation-sprint-2026-03-17-canonical.md
project: qmd
tags: [qmd, documentation, readme, help, mcp, bench, upstream-pr]
---

# QMD Documentation Sprint — Canonical PR Plan (Refreshed 2026-06-06)

**Supersedes**: `plan-documentation-sprint-2026-03-17-canonical.md`
**Goal**: Upstream PRs that improve qmd's documentation and help system.
**Audience**: New users, LLM agents, and community contributors.
**Strategy**: Doc-only first (low risk, easy merge), then code change (help system).
**Guiding principle**: *"qmd is no longer a CLI tool with a library — it is a library
with a CLI."* (established 2026-03-18). CLI and MCP are consumers of the `QMDStore`
interface. New documentation decisions apply this library-first lens.

> **Why this refresh exists.** The 2026-03-17 canonical plan was validated three times
> in March and then never implemented — the Phase 1 edits were planned but never
> committed. Re-validation on **2026-06-06** against current `dev` (which is
> byte-identical to `upstream/main` for README.md, docs/SYNTAX.md, and
> src/mcp/server.ts) found that (a) the core gaps still exist, (b) **three rows of
> the MCP parameter table are now factually wrong** because v2.5.3 changed behavior,
> and (c) several new undocumented features have landed. This plan corrects all of it.

---

## Branch / upstream state (verified 2026-06-06)

- `dev` is **14 commits ahead, 0 behind** `upstream/main` (tobi/qmd) and `origin/main`.
- All 14 dev-only commits are local **docs/chore** commits — none touch README.md,
  docs/SYNTAX.md, src/mcp/server.ts, or src/cli/qmd.ts.
- `git diff upstream/main..dev -- README.md docs/SYNTAX.md src/mcp/server.ts` is **empty**.
- Local `main` is stale (12 behind upstream); **do not base PRs on local `main`**.
- **Conclusion**: validate and author edits on `dev`; the doc diff is identical to
  what a PR against `upstream/main` would contain. Cut the PR branch from a clean
  point that excludes the 14 local docs commits (e.g. branch off `upstream/main`
  and cherry-pick, or rebase the doc edits onto `upstream/main`).
- Version at validation: **v2.5.3**, commit `b98c65b`.

---

## Re-validation results (2026-06-06)

### ✅ Still valid — Phase 1 genuinely undone

| Claim | Current state | Evidence |
|-------|---------------|----------|
| No `-c`/Collection Filtering section | Absent | grep README.md |
| `collection show/include/exclude/update-cmd` undocumented | Collection Mgmt covers only add/list/remove/rename/ls | README.md:537-558 |
| `--intent`, `--no-rerank`, `-C/--candidate-limit` missing from README Options | Confirmed absent (flags exist in CLI) | README.md:635-664; qmd.ts |
| `vector-search`/`deep-search` aliases undocumented | Search Modes table omits them | README.md:612; qmd.ts (aliases exist) |
| `--max-docs-per-batch`/`--max-batch-mb` undocumented | Embed section has no memory flags | README.md:560-575 |
| SYNTAX.md broken `q` parameter | Still present | docs/SYNTAX.md:136 |
| `buildInstructions` says singular `collection` | Still wrong; schema uses plural array | server.ts:122-123 vs server.ts:310 |
| All `--help` output identical | 100% identical MD5; now 97 lines (was 85) | live CLI |

### ⚠️ STALE — corrections required before implementing

The original plan's **MCP Tool Parameters table** must be corrected (v2.5.3 changes):

| Original plan row | Corrected current behavior | Evidence |
|-------------------|----------------------------|----------|
| `get` `lineNumbers` default **false** | default **true** | server.ts:378 |
| `multi_get` `lineNumbers` default **false** | default **true** | server.ts:453 |
| `get` colon syntax `path:line` | now `path:from:count` (e.g. `#abc:120:40`) | server.ts:383; CHANGELOG 2.5.3 |
| (no `rerank` row) | `query` has `rerank` boolean, default true | server.ts:314 |

Also already-fixed since March (drop from "missing" list):
- `--explain` is now in README Options (README.md:644) — but **still lacks sample output**.
- `--from` is now in README Options (Get options block).

### 🆕 New undocumented features (added to scope per 2026-06-06 decision)

| Feature | README? | Notes |
|---------|---------|-------|
| `qmd doctor` | No | Whole diagnostics command undocumented |
| `qmd init` | No | Local index init undocumented |
| `--full-path` | No | Swaps `qmd://` for on-disk paths (13 uses in source) |
| Line-numbered-by-default + `--no-line-numbers` | No | **Default behavior change** to get/multi-get |
| `get`/`docid` `:from:count` suffix | Partial (`:line` only) | New slicing syntax |
| `--format <kind>` | No | New output selector |
| Literal-path storage | CHANGELOG only | Affects path round-trip docs |

### 🆕 Bench (folded in per 2026-06-06 decision)

`qmd bench` behavior **unchanged since v2.1.0** (re-verified 2026-06-06): usage still
points to source path `src/bench/fixtures/example.json`, no `--example`, no collection
validation, no all-zero warning. Full analysis:
`docs/bench-discoverability-report-2026-04-07.md`. Doc-PR documents what exists; the
UX guardrails (collection validation, all-zero warning, `--example`/`--init`) are
**Phase 2 code work**.

---

## Phase 1: Documentation PR (README.md + docs/SYNTAX.md + 1-word server.ts fix)

**Title**: `docs: CLI reference, collection-flag semantics, MCP params, bench, and new commands`

No behavioral code changes. Pure markdown plus one string fix in server.ts.

### README.md changes

1. **Collection Filtering section** (after Options): `-c` is a global name lookup,
   works from any directory; multiple `-c` = OR; excluded collections skipped unless
   named; note multi-collection top-K caveat (#181/#217). Use `-n`/`--all` for more.
2. **Missing collection subcommands** (in Collection Management): `show`, `include`,
   `exclude`, `update-cmd` with examples.
3. **Missing search options** (in Options): `--intent`, `--no-rerank`,
   `-C/--candidate-limit`. Add a sample `--explain` output block.
4. **New commands & flags**: document `qmd doctor`, `qmd init`, `--full-path`,
   `--no-line-numbers` (note line-numbered-by-default), `:from:count` get suffix,
   `--format <kind>`.
5. **Embed memory flags**: `--max-docs-per-batch`, `--max-batch-mb`.
6. **Search aliases**: add `vector-search` (→ vsearch) and `deep-search` (→ query)
   to the Search Modes table.
7. **MCP Tool Parameters table** — use the CORRECTED values:
   - `query`: `searches` (array, required, first 2x weight), `collections` (string[],
     OR; singular `collection` silently ignored), `intent`, `limit` (10), `minScore`
     (0), `candidateLimit` (40), **`rerank` (boolean, default true)**.
   - `get`: `file` (path, `#docid`, or `path:from:count`), `fromLine`, `maxLines`,
     **`lineNumbers` (boolean, default true)**.
   - `multi_get`: `pattern`, `maxBytes` (10240), `maxLines`,
     **`lineNumbers` (boolean, default true)**.
   - Note: unknown params silently ignored; HTTP `/query`/`/search` now return
     `qmd://collection/path` URIs (#576).
8. **Benchmarking section** (after Index Maintenance): adapt the bench report's
   README block — setup (`collection add test/eval-docs` + `embed`), run, `--json`,
   backend table, score interpretation (bm25 ~0.50 / vector ~0.70 / hybrid+full ~1.00),
   custom fixture schema, query-type labels. **Packaging constraint (verified via
   `npm pack --dry-run`):** the example fixture (`src/bench/fixtures/example.json`)
   and test corpus (`test/eval-docs/`) are NOT in the published package (`files` ships
   `dist/bench/*.js` only). Frame the example as **git-checkout-only**; installed
   users must write a custom fixture against their own indexed collection. Use
   `qmd embed -c eval-docs` (not bare `qmd embed`) so setup doesn't re-embed the whole
   index. Shipping the fixture as package data is Phase 2 packaging work.

### docs/SYNTAX.md changes

1. **Remove broken `q` example** (line 134-140): the `query` tool and REST endpoint
   accept only `searches` (array). Replace with the structured `searches` form already
   shown just below it.
2. **Add Scoping section** (after Constraints): `-c` (CLI) / `collections` (MCP/SDK),
   by name, OR semantics, excluded-collection behavior.

### server.ts one-word fix

`buildInstructions` (server.ts:123): `scope with \`collection\` parameter`
→ `scope with \`collections\` parameter` (matches the plural array schema; the
singular form misleads agents into a silently-ignored param).

---

## Phase 1.25: `index.yml` configuration documentation (companion PR)

**Separate, focused companion PR** — keep Phase 1 reviewable rather than folding this
in. Rationale: the maintainer publicly called `index.yml` "under documented," so it
deserves its own spotlight; the content is large enough to stand alone; it has one
coherent theme (declarative configuration); and it references Phase 1 but does not
depend on it.

**Status (2026-06-06 continuity review):** A second review found the March 25
`index.yml` addendum had been silently dropped from the June plan — neither
implemented nor deferred. This phase captures it so the thread isn't lost. All
technical facts in the addendum were **re-verified against current `dev`/`upstream/main`
source on 2026-06-06** and still hold (schema fields at `collections.ts:28-33`,
`global_context` at :49, config-path precedence `QMD_CONFIG_DIR` → `XDG_CONFIG_HOME` →
`~/.config/qmd/` at :113-119, `example-index.yml` present in repo root).

**Full spec**: `docs/plan-index-yml-documentation-2026-03-25.md` (still accurate).

Scope of the companion PR:
- README **Configuration** section (placed *before* Collection Management) — `index.yml`
  location, the schema/fields table (`path`/`pattern`/`ignore`/`update`/
  `includeByDefault`/`context`), `global_context`, automatic update commands, ignore
  patterns (YAML-only, no CLI), config-location precedence, named indexes (`--index`).
- README **Environment Variables** additions: `XDG_CONFIG_HOME`, `QMD_CONFIG_DIR`
  (the section currently lists only `XDG_CACHE_HOME`).
- **`example-index.yml` overhaul** — bundled in this same PR (the README section links
  to it, so an underpowered example would read as unfinished). Keep it practical: show
  every field with concise comments, not a wall of prose.

Already handled in Phase 1 (do not repeat): the misleading `qmd update --pull` example
was **removed** from the README (the flag is parsed but never consumed —
`updateCollections()` ignores it; the real mechanism is a per-collection `update`
command). The companion PR should document that real mechanism fully. The underlying
`--pull` dead-code itself (wire it up or drop it from `--help`) is a **code** change —
note it in the companion PR body as a discovery, or file as a Phase 2 item.

---

## Phase 2: Progressive Help + Bench guardrails (src/cli/qmd.ts, src/bench/)

**File an upstream issue first** before writing code.

- Per-subcommand help (currently all `--help` emit the identical 97-line block).
  Top-level = command directory; per-command = relevant flags + 1-2 examples; EBNF
  grammar lives only in `query --help`. Add `--intent`/`--from` to relevant help.
- Bench guardrails: validate fixture collection exists, warn on all-zero results,
  add `qmd bench --example` (print fixture JSON) so npm users don't need the source
  path. Per `bench-discoverability-report-2026-04-07.md`.

---

## Phase 1.5: Library-first framing (deferred — placeholder only)

**After Phase 1 lands upstream, reassess whether the README should lead with
`QMDStore`/SDK as the primary product and present the CLI/MCP as consumers.** Do not
start the design work until there is maintainer feedback on Phase 1 — drafting the
reframe now would add another planning artifact before Phase 1 is even reviewed.

---

## Upstream PR strategy

- Phase 1: file the docs PR directly; reference issues #25, #181, #217, #372, #520,
  #576 as motivation. Keep diff to README.md + docs/SYNTAX.md + the one-word
  server.ts fix.
- Phase 2: file an issue first (cite clig.dev, uv/cargo/gh precedent); wait for
  maintainer signal before the code PR.
- **Out of scope** for these PRs: bug fixes (hyphen handling, multi-collection
  filtering), new features/behavior, architecture diagrams.
