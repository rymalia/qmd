<!--
LOCAL STAGING ARTIFACT — not for upstream. This is the PR body for the Phase 1.25
index.yml companion PR. Use with:
  gh pr create --repo tobi/qmd --base main --head rymalia:docs/qmd-index-yml-config \
    --title "docs: document the index.yml configuration schema" \
    --body-file docs/pr-body-phase-1.25-index-yml.md
Do NOT `git add` this file onto the PR branch. Full spec:
docs/plan-index-yml-documentation-2026-03-25.md

RE-CHECK BEFORE FILING — source line numbers drift. Verified against `dev` on
2026-06-08:
  src/collections.ts — Collection interface :27-34, global_context :49,
    editor_uri/editor_uri_template :50-51, models :39-53, config-dir precedence
    (QMD_CONFIG_DIR / XDG_CONFIG_HOME / default) :114-121, global file is `.yml`
    :125, project-local `.yaml`/`.yml` precedence :138-143
  src/cli/qmd.ts — `qmd init` seeds resolved models: :421, `update` exec via
    bash -c in col.pwd + non-zero abort :683-717
  src/store.ts — `ignore` merge with built-in excludes :1284-1296
The public PR body below intentionally omits line numbers so it can't go stale.
-->

## Summary

`index.yml` is qmd's declarative collection config, including the per-collection
`update` command that runs automatically before `qmd update` re-indexes a collection.
That automatic update hook has been called out as under-documented, yet the README
currently mentions `index.yml` zero times: not the update command, not the file path,
not the schema, and not the fact that users can edit and version-control it directly.

This PR documents the full `index.yml` configuration system.

Documenting the schema also reduces duplicate work: because model resolution from
`index.yml` wasn't documented, contributors repeatedly re-submitted fixes for behavior
that already ships (#502, #559, #564), and users keep requesting config that already
exists (#645 exclusions, #678 model config).

Doc-only follow-up to the CLI/MCP reference PR (#715); references it but does not
depend on it.

## What's documented

**README.md — new "Configuring `index.yml`" section** near collection management:
- File location and editability: `~/.config/qmd/index.yml`, the `XDG_CONFIG_HOME` and
  `QMD_CONFIG_DIR` overrides, named indexes (`{name}.yml`), and project-local
  `.qmd/index.yml` (`.qmd/index.yaml` also accepted) created by `qmd init`.
- An annotated YAML example plus a key-reference table for every key:
  `global_context`, `editor_uri`, the `models.embed`/`rerank`/`generate` overrides,
  and per-collection `path`/`pattern`/`ignore`/`update`/`includeByDefault`/`context`.
- **Automatic update commands** (own subsection): the per-collection `update` command
  runs via `bash -c` in the collection's own directory before re-indexing; a non-zero
  exit aborts the whole `qmd update` run; set/clear with `qmd collection update-cmd`.
  This is the real mechanism #715 redirected users to when it removed the no-op
  `qmd update --pull` example.
- `ignore` is documented as **YAML-only** (no CLI command) and **additive** with the
  built-in exclusions (`node_modules`, `.git`, `.cache`, `vendor`, `dist`, `build`),
  which cannot be un-ignored.
- "Editing ≠ re-indexing" note (run `qmd update` after `path`/`pattern`/`ignore`
  changes, `qmd embed` after `models.embed`).

**README.md — Environment Variables**: added `XDG_CONFIG_HOME` and `QMD_CONFIG_DIR`.
**README.md — Model Configuration**: now points to the `models:` / `QMD_EMBED_MODEL`
override path instead of implying models are editable only in source.

**example-index.yml** — rewritten from three near-identical collections into a
fully-commented starter template where each collection demonstrates a distinct feature
(hierarchical context, auto-`update`, `ignore` patterns, non-markdown globs,
`includeByDefault: false`, and an all-fields example). The README links to it. The
`models:` block is shown commented with placeholder URIs so the template can't become
a second source of truth for the current model defaults.

## Out of scope

- No code changes, no schema changes, no new keys, no validation. The `qmd update
  --pull` dead-code (wire it up or remove from `--help`) remains a separate follow-up.

## Verification

- Every documented key verified against `src/collections.ts` and its consumers
  (`src/cli/qmd.ts`, `src/store.ts`).
- `example-index.yml` parses as valid YAML and is present in the repo root.
