## Summary

`CLAUDE.md`'s command reference has drifted from the CLI — its last substantive
update was the March AST-chunking change. Most notably it documents `qmd update
--pull` as "git pull first," but `--pull` is parsed and never consumed by
`updateCollections()`, so it's a no-op. #715 already removed this same claim from
the README; this brings `CLAUDE.md` in line and refreshes the rest of the list.

Scope is deliberately a cheat-sheet refresh: remove wrong guidance, add commands and
flags an agent is likely to use. No behavior changes, and nothing stylistic touched.

## Changes

- **Fix `--pull`**: it's dead code. The real pre-reindex mechanism is a per-collection
  `update` hook (`qmd collection update-cmd`), which runs automatically on `qmd update`.
- **Add commands that now exist**: `qmd init`, `qmd doctor`, `qmd bench`.
- **Add collection subcommands**: `show`, `update-cmd`, `include`, `exclude`.
- **Lead output options with `--format <kind>`** (`cli|json|csv|md|xml|files`); the bare
  `--json`/`--csv`/`--md`/`--xml`/`--files` booleans are noted as legacy aliases.
- **Document `qmd get <file>[:from[:count]]`** line-range syntax and add the
  `--intent`, `--no-rerank`, `--full-path`, and `--no-line-numbers` flags.

## Left untouched

The Bun-first guidance, the "Do NOT run automatically" / "Do NOT compile" sections,
the Architecture notes, and the Releasing section — all still accurate.

## Verification

Every command and flag was checked against `src/cli/qmd.ts` in current source.
`--pull` is confirmed parsed but never read by `updateCollections()`. The
`vector-search`/`deep-search` aliases were intentionally left undocumented — the
source labels them "undocumented alias."
