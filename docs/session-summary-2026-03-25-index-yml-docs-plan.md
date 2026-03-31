---
date: 2026-03-25
time: "9:31 PM PDT – 10:34 PM PDT"
project: qmd
branch: dev
---

# Session Summary: index.yml Documentation Plan

## Overview

Investigated tobi's tweet about qmd's `index.yml` being "under documented," reverse-engineered the full feature set from source code, and produced a standalone documentation plan covering the entire `index.yml` configuration system — including a proposed overhaul of `example-index.yml`.

## Key Decisions Made

- **index.yml documentation as a dedicated plan**: Rather than folding this into the existing canonical documentation sprint plan, created a standalone addendum (`docs/plan-index-yml-documentation-2026-03-25.md`) that can be incorporated into the same upstream PR or submitted as a companion.
- **example-index.yml needs a full overhaul**: The current file shows 3 identical-structure collections with only `path`, `pattern`, and `context`. Proposed replacement has 6 collections, each demonstrating a different feature (`update`, `ignore`, `includeByDefault`, non-markdown patterns), with heading comments that teach the schema by example.
- **New "Configuration" section goes before "Collection Management" in README**: Establishes that collections live in a YAML file before showing CLI commands that write to it, creating a coherent narrative.

## Changes Made

| Change | Detail |
|--------|--------|
| **Created index.yml documentation plan** | `docs/plan-index-yml-documentation-2026-03-25.md` — full plan covering the gap, evidence base, proposed README additions, example-index.yml overhaul, and integration strategy with the canonical sprint plan |

## Research Performed

- **Dev branch status audit**: Confirmed `dev` is fully caught up with both `main` and `upstream/main` (zero commits behind). The rebase prerequisite blocker from the canonical plan is resolved.
- **Fetched davidgasquez/dotfiles commit**: Analyzed the tweet's context — David created a project-local `.qmd/index.yml` with a shell wrapper using `QMD_CONFIG_DIR` and `INDEX_PATH` env vars for per-project indexes.
- **Full source audit of index.yml features**: Traced every field in the `Collection` interface through `src/collections.ts`, `src/cli/qmd.ts`, `src/store.ts`, and test files. Mapped CLI commands to YAML fields, identified documentation gaps, and verified behavior.
- **Discovered `--pull` is dead code**: `qmd update --pull` is documented in README and `--help` but the parsed flag value is never consumed. The `update` field in `index.yml` is the actual mechanism for running pre-update commands.
- **Discovered `ignore` has no CLI**: Can only be set by editing YAML directly, making documentation of the file essential.
- **Audited `example-index.yml`**: Confirmed it demonstrates none of `update`, `ignore`, `includeByDefault`, non-markdown patterns, or detailed comments.

## Testing Performed

- Verified local `~/.config/qmd/index.yml` exists and is active
- Confirmed `upstream/main` fetch returned no new commits
- Validated all source code line references via direct file reads

## Summary Statistics

- 1 planning document created (~320 lines)
- 0 code changes
- 11 source files audited for index.yml references
- 1 external commit fetched and analyzed (davidgasquez/dotfiles)
- 6 undocumented features catalogued (update, ignore, includeByDefault, QMD_CONFIG_DIR, XDG_CONFIG_HOME, --index)

## Unfinished Work

- **Phase 1 PR not yet submitted**: The canonical plan's README/SYNTAX.md edits exist locally on `dev` but haven't been submitted upstream. The index.yml additions should be incorporated before or alongside that PR.
- **Validation checklist in the plan**: All proposed additions need empirical verification against a fresh install before PR submission (update command cwd, ignore patterns, --index named indexes, QMD_CONFIG_DIR override).
- **`--pull` flag decision**: Either implement it or remove from docs/help. Should be raised as a separate issue or noted in the PR body.
- **`findContextForPath()` divergence**: `collections.ts` returns single best match; `store.ts` returns all matching prefixes. Code consistency issue — not a docs task, but worth filing separately.
