---
session_id: 91eb93db-a4d0-4fd5-8bcb-00480fa9f6a1
date: 2026-06-07
time: "3:06 PM PDT – 6:03 PM PDT"
project: qmd
branch: dev
related_pr: 715
---

# Session Summary: Upstream Direction Analysis + `index.yml` Docs

## Overview

A two-part session: (1) a strategic analysis of the upstream `tobi/qmd` repo —
mining issues, PRs, and maintainer comments to assess project direction and judge
whether our contribution plans are aligned or outliers; and (2) acting on that
analysis by drafting the next planned doc contribution (the `index.yml`
configuration schema for the README), source-verified against the codebase.

## Key Decisions Made

- **We are not outliers.** Two independent analyses converged: our #715/#716
  direction fits the dominant trajectory (local-first agent retrieval; CLI/MCP/HTTP
  ergonomics; reliability; config clarity). Our chosen lane — docs/discoverability —
  is *underserved* and mirrors the maintainer's own current priorities.
- **Committed a 4-step contribution sequence:** (1) **Phase 1** — #715 in flight →
  await signal; (2) **Phase 1.25** — `index.yml` docs PR next (concrete,
  source-verifiable, independently demanded by #645/#678); (3) **Phase 2** — #716
  code, framed as a "discoverability guardrail for already-shipped commands," kept
  boring and test-backed; (4) **Phase 1.5** — library-first README reframe **held /
  reactive only** — positioning is taste, and a maintainer-as-author owns voice.
  (Phase labels match the authoritative roadmap in
  `docs/plan-documentation-sprint-2026-06-06-canonical.md`, which also tracks an
  independent **Phase 1.75** — HTTP server REST/ops docs — committed separately
  today.)
- **#716 must be shaped to dodge the batch-and-supersede risk:** no new help
  framework unless the parser forces it; `--help` tests must be *read-only assertions
  over current output* (a test demanding flag registration is a framework in
  disguise); compact output; PR body leads with the regression-style problem.
- **Adopted a standing DO-NOT list** derived from verified maintainer rejections
  (see Discoveries) to keep all future proposals inside the merge envelope.
- **`index.yml` docs framed as a duplicate-PR deflector**, not docs-for-docs' sake —
  the framing that clears tobi's explicit #699 "docs-only is not good enough" bar.

## Changes Made

| Change | Detail |
|--------|--------|
| **Upstream direction analysis doc** | Created `docs/upstream-direction-analysis-2026-06-07.md` — consolidated assessment (maintainer operating model, what merges vs. is dismissed, community landscape, our standing, committed sequence, DO-NOT list). QMD-searchable frontmatter; all citations verified. |
| **New "Configuring `index.yml`" README section** | `README.md` — full schema (annotated YAML + key-reference table) for `global_context`, `editor_uri`, `models.*`, and per-collection `path`/`pattern`/`ignore`/`update`/`includeByDefault`/`context`; file-location rules; "edit ≠ re-index" note. |
| **Env-var table correctness** | `README.md` — added `XDG_CONFIG_HOME` and `QMD_CONFIG_DIR` (source-true, previously undocumented). |
| **Model Configuration cross-ref** | `README.md` — corrected the misleading "configured in `src/llm.ts`" line; added the `index.yml` `models:` / `QMD_EMBED_MODEL` override path. |
| **Changelog entry** | `CHANGELOG.md` — `[Unreleased] → Documentation` entry with the deflector rationale and issue refs (#502/#559/#564/#645/#678). |

## Research Performed

- **Upstream repo surveyed end-to-end** (github.com/tobi/qmd, 26K⭐): all 92 issues
  and ~120 PRs from Apr 1 – Jun 7 2026; merged vs. closed-unmerged vs. open split;
  CHANGELOG arc v0.1.0 → Unreleased.
- **Maintainer directional comments verified verbatim** on 12+ threads (#516, #560,
  #649, #481, #631, #587, #610, #699, #517, #620, plus the #559/#525/#564
  supersession cluster and the #470/#449 enthusiasm quotes).
- **Cross-checked a second analyst's report** — all load-bearing citations
  re-verified against the GitHub API; it independently converged and surfaced the
  decision-relevant #699 "docs-only not good enough" signal.
- **Source-verified the entire `index.yml` schema** before writing a line: read
  `src/collections.ts` in full and traced consumption of every key through
  `src/index.ts`, `src/store.ts`, `src/cli/qmd.ts`, `src/llm.ts`; confirmed exact
  default model URIs, file-location rules, and `qmd init` seeding behavior.

## Summary Statistics

- **Upstream items reviewed:** ~92 issues + ~120 PRs (2-month window)
- **Maintainer comment threads verified:** 12+
- **Source files audited for schema verification:** 5
  (`collections.ts`, `index.ts`, `store.ts`, `cli/qmd.ts`, `llm.ts`)
- **Docs/files changed:** 3 (`README.md`, `CHANGELOG.md`, new analysis doc) + new
  session summary
- **`index.yml` keys documented:** 9, all source-verified as consumed

## Discoveries / Handoff Notes

- **Maintainer operating model = author, not curator.** tobi re-implements
  community-reported fixes in his own batch stacks (e.g. #636 +1,200) and closes the
  originals as "superseded." Small (<60-line) correct local-ethos fixes merge fast;
  large features and anything expanding the product surface do not.
- **Verified DO-NOT list** (each backed by a tobi quote): no Python/second-language
  SDK (#560); no MLX/alt runtime (#649); no knowledge graph (#481); no ad-hoc
  text-compare command (#631); no write-side MCP tools (#587); no remote/cloud
  backend except as a contribution to the **#620** umbrella (one OpenAI-compatible
  design, VCR-style tests); no docs-only fix that leaves users stranded (#699); no
  unsolicited README/identity reframing.
- **Remote backends are a ~10-PR graveyard** — the single largest rejected theme.
  #620 is the *only* sanctioned channel.
- **`qmd init` seeds `.qmd/index.yml` with a resolved `models:` block**
  (`cli/qmd.ts:421`) — concrete proof the `models:` key is first-class, useful as
  anti-pushback evidence in the docs PR.
- **Global config is `.yml` only; project-local accepts `.yaml` or `.yml`**
  (`collections.ts:139-143`) — a real correctness gotcha now documented.
- The full analysis (with citation links) lives in
  `docs/upstream-direction-analysis-2026-06-07.md` and is QMD-searchable.

## Issues & PRs

- **PR #715** (ours, OPEN): https://github.com/tobi/qmd/pull/715 — CLI/MCP/bench
  docs. In flight; awaiting maintainer signal before opening the follow-ups.
- **Issue #716** (ours, OPEN): https://github.com/tobi/qmd/issues/716 — per-subcommand
  `--help` + bench guardrails (code; step 3 of the sequence).
- **Umbrella #620**: https://github.com/tobi/qmd/issues/620 — the only sanctioned
  path for remote-backend work, if ever pursued.

## Unfinished Work / Next Steps

- **Commit** this session's staged changes (`README.md`, `CHANGELOG.md`, the analysis
  doc, this summary) — user will perform the commit per the no-auto-commit rule.
- **Hold the Phase 1.25 `index.yml` docs PR** until #715 (Phase 1) gets a maintainer
  signal. Ready-to-use PR body below.
- **Phase 2 (#716)** code work comes after #715 engagement; keep it boring and
  test-backed.
- **Phase 1.5** library-first reframe remains gated on maintainer feedback — do not
  lead with it.

## Ready-to-Use Draft: Phase 1.25 `index.yml` Docs PR

The README/CHANGELOG changes for this are **already committed** in this session; this
is the PR body to open once #715 gets a maintainer signal. Copy verbatim.

**Title:** `docs: document the index.yml configuration schema`

```markdown
## Summary
`index.yml` is the single source of truth for collections, contexts, per-collection
update hooks, exclusions, and model overrides — but its format is undocumented in the
README. The CLI commands that write it (`collection add`, `collection update-cmd`,
`collection include/exclude`, `context add`) are documented; the file they produce,
and the keys you can only set by editing it directly (`ignore`, `models`,
`global_context`, `editor_uri`), are not.

This is a recurring source of duplicate work: because model resolution from
`index.yml` wasn't documented, contributors repeatedly re-submitted fixes for
behavior that already ships (#502, #559, #564), and users keep requesting config that
already exists (#645 exclusions, #678 model config). Documenting the schema deflects
that.

## What's added
- A **"Configuring `index.yml`"** README section: annotated YAML example + a
  key-reference table, file-location rules (`XDG_CONFIG_HOME`/`QMD_CONFIG_DIR`, named
  `{name}.yml`, project-local `.qmd/index.yml`), and an "editing ≠ re-indexing" note.
- `XDG_CONFIG_HOME` and `QMD_CONFIG_DIR` added to the env-var table.
- Model Configuration section now points to the `models:` / `QMD_EMBED_MODEL` override
  path instead of implying models are only editable in source.

## Scope
Docs only — no schema changes, no new keys, no validation. Every documented key is
verified against `src/collections.ts` and its consumers.

## Related
Documents behavior behind #645, #678; reduces duplicate-PR pressure seen in
#502/#559/#564. Companion to #715.
```
