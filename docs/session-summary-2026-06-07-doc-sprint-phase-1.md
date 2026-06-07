---
session_id: 7851355b-6522-456b-a123-6e3fae0a0478
date: 2026-06-07
time: "2026-06-06 08:05 PM PDT – 2026-06-07 02:27 PM PDT"
resumed: "2026-06-07 01:29 AM PDT, 2026-06-07 01:31 PM PDT"
project: qmd
branch: docs/qmd-reference-phase-1
related_pr: 715
---

# Session Summary: Documentation Sprint — Phase 1 Shipped

## Overview

Revived the QMD documentation sprint that had stalled since March/April 2026,
re-validated the entire plan against current source, implemented Phase 1 in full
(CLI reference, MCP params, bench docs, new commands), survived two independent
dev-team review rounds, and shipped it as upstream PR **#715** plus a companion
Phase 2 issue **#716**. The remaining phases are planned, staged, and gated on
maintainer signal.

## How We Got Here

The user remembered passionately working on README/`--help` improvements "way back
in April" and wanted to find and finish that work. Investigation surfaced a rich but
**unfinished** trail: a thrice-validated canonical plan from 2026-03-17, an onboarding
feedback doc (v2.0.1), and an April bench discoverability report. Critically, the
project memory claimed Phase 1 was "complete as of 2026-03-18" — but the working tree
was clean and contained **none** of those edits. The plan had been designed and
validated, then never implemented (or lost in a rebase). We were effectively starting
Phase 1 from scratch, but with a strong spec in hand.

## Key Decisions Made

| Decision | Rationale |
|----------|-----------|
| **Re-validate before implementing** | The plan was ~80 days old against a fast-moving upstream. Re-validation caught 3 now-incorrect MCP-table rows that would have shipped wrong defaults. |
| **Target upstream `tobi/qmd`** | Kept the original upstream-PR framing; docs PR first (low-friction), code PR (help system) later. |
| **Full Phase 1 scope incl. bench** | User chose the most complete option — folded the April bench report's findings into the docs. |
| **`index.yml` as a separate companion PR (Phase 1.25)** | A continuity review found the March 25 `index.yml` addendum had been silently dropped. Rather than balloon #715, carved it into a focused companion PR — the maintainer publicly called `index.yml` under-documented, so it deserves its own spotlight. |
| **Bundle `example-index.yml` rewrite with Phase 1.25** | README config docs referencing an underpowered example file would read as unfinished. |
| **Defer Phase 1.5 (library-first reframe)** | Don't design the README restructure until there's maintainer feedback on Phase 1. |
| **Drift-proof the Phase 1.25 PR body** | Removed exact source line numbers from the public body (kept them in a local-only HTML comment with a re-check reminder) so the body can't go stale before filing. |

## Changes Made

| Change | Detail |
|--------|--------|
| **Refreshed canonical plan** | `docs/plan-documentation-sprint-2026-06-06-canonical.md` supersedes the March version; records branch state, re-validation results, corrected scope, and Phases 1/1.25/1.5/2. |
| **README.md** | +192 lines: collection filtering, `show`/`include`/`exclude`/`update-cmd`, `--intent`/`--no-rerank`/`-C`/`--full-path`, `--format <kind>` (legacy booleans as aliases), `vector-search`/`deep-search` aliases, embed memory flags, real `--explain` trace, MCP tool parameter table, `qmd doctor`/`init`, `get :from:count` + `--no-line-numbers`, Benchmarking section. Removed the misleading `qmd update --pull` example. |
| **docs/SYNTAX.md** | Removed the non-existent `q` MCP parameter example; added a Scoping section (with valid JSON). |
| **src/mcp/server.ts** | `buildInstructions`: singular `collection` → plural `collections` (matches schema); `get` instruction now documents the full `file.md:from:count` range suffix. |
| **CHANGELOG.md** | Documentation + Fixed entries under `[Unreleased]`. |
| **Phase 2 issue draft** | `docs/issue-draft-progressive-help-bench-guardrails.md`. |
| **Phase 1.25 PR body** | `docs/pr-body-phase-1.25-index-yml.md` (local staging artifact, drift-proofed). |
| **Project memory** | Corrected the false "complete in March" note; recorded the 2026-06-06 implementation and Phase 1.25. |

## Research / Validation Performed

- **Plan re-validation** against current `dev` (= byte-identical to `upstream/main`
  for all touched files): grepped CLI flags, subcommand dispatch, README sections;
  confirmed every "still missing" claim and flagged the stale ones.
- **MCP schema verified live** by reading `server.ts:300-318, 375-381` before writing
  the parameter table — caught `lineNumbers` default flip (false→true in v2.5.3), the
  new `rerank` param, and the `:from:count` suffix.
- **Captured a real `--explain` trace** (`qmd query --json --explain`) rather than
  fabricating the sample JSON.
- **`npm pack --dry-run`** proved the bench fixture/test corpus are NOT shipped —
  corrected README from "shipped example fixture" to git-checkout-only framing.
- **Confirmed `--pull` is dead code**: `updateCollections()` takes no args and never
  reads the parsed flag.
- **Two independent review rounds** (dev-team consultant): round 1 found 6 issues
  (npm fixture path, `embed -c`, invalid JSON-in-`json`-fence, `--format`, `get`
  syntax, framing); round 2 (continuity) found 3 (dropped `index.yml` thread, `--pull`
  no-op, stale `get` slicing in server.ts). All addressed and re-reviewed clean.

## Summary Statistics

- **PR #715**: 4 product files, **232 insertions / 22 deletions**
- **Issue #716** filed (Phase 2)
- **9 review findings** addressed across 2 rounds
- **3 stale claims** caught and corrected vs the March plan
- Planning artifacts: 1 refreshed canonical plan, 1 issue draft, 1 staged PR body
- Phases defined: **4** (1 shipped, 1.25 staged, 1.5 deferred, 2 issue-filed)

## Discoveries / Handoff Notes

- **`qmd update --pull` is dead code** — parsed at `qmd.ts:2868`, never consumed;
  `updateCollections()` ignores it. Real pre-reindex mechanism is the per-collection
  YAML `update` field / `qmd collection update-cmd`. Wiring it up or removing from
  `--help` is a code follow-up (noted in #715 body, out of scope for the doc PR).
- **v2.5.3 flipped MCP `lineNumbers` to default `true`** for `get`/`multi_get`, added
  a `query` `rerank` param, and changed `get` colon syntax to `:from:count`. The old
  plan's param table predated all three.
- **`index.yml` is under-documented per the maintainer** (tobi's public statement).
  Full spec preserved in `docs/plan-index-yml-documentation-2026-03-25.md`; all its
  facts re-verified against current `src/collections.ts` on 2026-06-06.
- **Bench fixture (`src/bench/fixtures/example.json`) + `test/eval-docs/` are not in
  the npm package** (`files` ships only `dist/bench/*.js`). Docs must frame the example
  as git-checkout-only.
- **Branch topology**: work committed as `ec69677` on `dev`; cherry-picked to
  `7488fe8` on `docs/qmd-reference-phase-1` (cut off `upstream/main`); pushed to
  `origin` (the fork), PR opened against `tobi/qmd:main`. The PR branch tracks
  `origin/...` after the explicit `git push -u origin`.

## Current State

- **PR #715** open at https://github.com/tobi/qmd/pull/715 — awaiting review.
- **Issue #716** open at https://github.com/tobi/qmd/issues/716 — awaiting signal.
- Checked-out branch: `docs/qmd-reference-phase-1`.
- Untracked local artifacts in `docs/` (keep out of upstream PRs):
  `plan-documentation-sprint-2026-06-06-canonical.md`,
  `issue-draft-progressive-help-bench-guardrails.md`,
  `pr-body-phase-1.25-index-yml.md`, `replay-7851355b.md`.
- Local MCP daemon **not** refreshed — the `server.ts` instruction fixes are committed
  but won't show in local agent sessions until a daemon restart (optional; no PR impact).

## Unfinished Work / Next Steps

1. **Await review on #715.** If tobi engages → cut `docs/qmd-index-yml-config` off
   `upstream/main` and implement Phase 1.25 from the March 25 spec, filing with the
   staged (drift-proofed) PR body. **Re-grep the `src/collections.ts` line numbers
   first** (reminder lives in the PR-body HTML comment).
2. **Await signal on #716** before starting the Phase 2 code (progressive `--help` +
   bench guardrails).
3. **Phase 1.5** (library-first README reframe) stays deferred until there's
   maintainer reception to gauge.
4. Optional: refresh the local MCP daemon to pick up the `server.ts` instruction fixes.

## Issues & PRs

- **PR #715** — docs: CLI reference, collection-flag semantics, MCP params, bench,
  new commands — https://github.com/tobi/qmd/pull/715
- **Issue #716** — Per-subcommand `--help` and bench discoverability guardrails —
  https://github.com/tobi/qmd/issues/716

## Secondary Reviewer Notes

The most important thing this session preserved was not just the old documentation
plan, but the discipline around it. The March/April artifacts were high-quality, but
they were not safe to ship unchanged: upstream had moved, defaults had changed, and
some planned claims had become false. The useful pattern was: read the plan, read the
source, compare the staged diff to both, and treat every confident sentence in the
README as something that needs a current source anchor.

Three review themes stood out:

- **Continuity matters as much as correctness.** The dropped `index.yml` thread was
  not a typo in the PR; it was a planning continuity break. Catching it before filing
  #715 let Phase 1 stay focused without losing the maintainer-requested config docs.
- **Documentation can expose product bugs.** The `--pull` finding is a good example:
  the docs review surfaced dead CLI behavior. The right response was not to sneak in a
  code fix, but to remove misleading docs, name the follow-up, and keep the PR's
  behavioral scope clean.
- **Agent-facing docs need extra precision.** The singular `collection` MCP hint was a
  one-word bug with large practical impact because Zod silently strips unknown params.
  Likewise, `lineNumbers`, `rerank`, and `:from:count` needed exact current defaults
  because LLM clients will copy those details directly.

The final shape feels right: #715 is broad but still coherent, #716 captures the code
work before implementation starts, Phase 1.25 gives `index.yml` the spotlight it
deserves, and Phase 1.5 remains gated on maintainer appetite. That sequencing is the
main strategic outcome of the session.
