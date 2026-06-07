<!--
LOCAL STAGING ARTIFACT — not for upstream. This is the PR body for the Phase 1.25
index.yml companion PR. Use with:
  gh pr create --repo tobi/qmd --base main --head rymalia:docs/qmd-index-yml-config \
    --title "docs: document index.yml configuration" \
    --body-file docs/pr-body-phase-1.25-index-yml.md
Do NOT `git add` this file onto the PR branch. Full spec:
docs/plan-index-yml-documentation-2026-03-25.md

RE-CHECK BEFORE FILING — source line numbers drift. As of 2026-06-07 the relevant
defs in src/collections.ts were: schema fields :28-33, global_context :49,
config-path precedence (QMD_CONFIG_DIR / XDG_CONFIG_HOME / default) :113-119.
Re-grep these before implementing. The public PR body above intentionally omits
line numbers so it can't go stale.
-->

## Summary

Documents qmd's declarative configuration file, `~/.config/qmd/index.yml`. The
maintainer has publicly noted this feature is under-documented — the README currently
mentions `index.yml` zero times, even though it is the single source of truth for
every collection's path, glob, ignore rules, contexts, update command, and query
inclusion. Users learn about collections only through CLI commands, with no awareness
that those commands read/write a YAML file they can edit directly (and some fields,
like `ignore`, have no CLI at all).

Doc-only follow-up to the CLI/MCP reference PR (#715); references it but does not
depend on it.

## What's documented

**README.md — new Configuration section** (placed before Collection Management, so
readers know collections live in YAML before seeing the CLI that edits it):
- File location `~/.config/qmd/index.yml` and that it's human-editable.
- Collection fields table: `path`, `pattern`, `ignore` (YAML-only — no CLI),
  `update`, `includeByDefault`, `context`.
- Top-level `global_context`.
- Automatic update commands — the real "run `git pull` before re-indexing" mechanism
  (runs per-collection via `bash -c`; non-zero exit aborts the update). This is the
  mechanism #715 redirected users to when it removed the no-op `qmd update --pull`
  example.
- Config-location precedence: `QMD_CONFIG_DIR` → `XDG_CONFIG_HOME` → `~/.config/qmd/`.
- Named indexes (`--index` → separate `.yml` + `.sqlite`).

**README.md — Environment Variables** additions: `XDG_CONFIG_HOME`, `QMD_CONFIG_DIR`
(the section currently lists only `XDG_CACHE_HOME`).

**example-index.yml** — rewritten to demonstrate the full schema: every field shown
with concise inline comments (the README section links to it, so an underpowered
example would read as unfinished). Practical, not ornate.

## Out of scope

- No code changes. In particular, the `qmd update --pull` dead-code (wire it up or
  remove from `--help`) is a separate follow-up, not part of this doc PR.

## Verification

- Schema, fields, and config-location precedence verified against `src/collections.ts`
- `example-index.yml` confirmed present in repo root and accurate

