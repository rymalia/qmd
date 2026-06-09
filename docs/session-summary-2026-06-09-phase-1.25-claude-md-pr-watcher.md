---
session_id: 5b6b7101-4582-4615-9ac0-3531a2411bc2
date: 2026-06-09
time: "2026-06-08 06:04 PM PDT – 2026-06-09 12:49 AM PDT"
project: qmd
branch: dev
related_pr: "718, 719"
---

# Session Summary: Phase 1.25 + CLAUDE.md Shipped, Cloud PR-Watcher Stood Up

## Overview

Caught up on the doc-sprint state, confirmed **PR #715 merged** (the maintainer
signal we were gated on), then shipped two more upstream docs PRs — **#718** (the
`index.yml` configuration schema, strengthened) and **#719** (a CLAUDE.md
command-reference refresh) — and finally stood up a **cloud cron routine** that
watches #716/#718/#719 and pushes to the user's iPhone on maintainer activity
(mobile push verified working end-to-end).

The strategic headline is that #715 was not just "a docs PR got merged." It was the
first real test of the upstream-direction thesis from 2026-06-07: docs/discoverability
is an underserved, mergeable lane even in a maintainer-as-author project where larger
community work is often reimplemented. A silent merge by tobi, with no requested
changes, turned Phase 1.25 from a speculative follow-up into a timely continuation.

## How We Got Here

The previous two sessions (2026-06-07) shipped Phase 1 (#715) and staged Phase 1.25
(`index.yml` docs) plus a strategic upstream-direction analysis. This session opened
by catching up on those four artifacts and checking PR/issue status — at which point
we discovered **#715 had been silently merged by tobi within ~a day**, clearing the
"await maintainer signal" gate and validating the whole docs/discoverability lane.

## Key Decisions Made

| Decision | Rationale |
|----------|-----------|
| **Strengthen Phase 1.25 before filing** | The committed README draft had flattened the marquee feature (the automatic `update` hook tobi *publicly* called under-documented) into a single table row, and had silently dropped the `example-index.yml` overhaul from the March plan. Restored both before #718 went out. |
| **`example-index.yml`: keep `models:` commented** | Live GGUF URIs in the template would become a second source of truth that drifts from the README/defaults. Placeholder + pointer instead. |
| **PR #718 body leads with the maintainer hook** | Reframed from "duplicate-PR deflector" to "tobi called the automatic update feature under-documented," with the deflector argument demoted to support. Phrased as maintainer-aligned docs, not "because of a tweet." |
| **CLAUDE.md = correctness + missing-commands PR** | It's in upstream and byte-identical to ours, last touched 2026-03-22, and carries the *same `--pull` lie* #715 removed from the README. Fixed the lie + added real commands; left Bun stance / DO-NOT / Architecture / Releasing untouched (voice, not correctness). |
| **CLAUDE.md PR = single file, no CHANGELOG** | A CHANGELOG line on `dev` would tangle with #718's still-open CHANGELOG changes at cherry-pick time; CLAUDE.md is contributor guidance, not user-facing software. |
| **Hold Phase 1.75; don't flood** | Three docs PRs (1 merged, 2 open) is a sane ceiling. Let #718/#719 land before filing the HTTP REST/ops docs. |
| **Stand up a cloud PR-watcher** | Both PRs + #716 now wait on one external human; a 3-hourly cloud routine with mobile push beats manual polling and survives the laptop being closed. |

## Quality / Process Lessons

- **The recurring failure mode is not bad planning — it is under-carrying good plans
  into implementation.** Phase 1 had been designed but not actually shipped; Phase
  1.25 repeated the smaller version by compressing the README correctly but dropping
  `example-index.yml`. The durable habit is: compare the implementation back to the
  canonical plan and ask what deliverables silently disappeared.
- **Compression was still the right instinct.** The fix was not to restore the March
  plan's sprawling README layout wholesale. It was to restore depth only where it
  earned its keep: the automatic `update` hook tobi had called under-documented, plus
  the starter template that teaches the schema by example.
- **Docs need source anchors, not memory anchors.** Every confident sentence about
  `index.yml`, `--pull`, `ignore`, `get` ranges, output formats, and CLAUDE.md command
  examples was checked against current source before filing. That prevented both stale
  March facts and new accidental errors.
- **Agent-facing docs have the same correctness bar as user docs, but not the same
  breadth.** `CLAUDE.md` deserved a correctness refresh because it contained the same
  `--pull` falsehood #715 removed from the README; it did not deserve a voice rewrite
  or a second README.
- **The clean-branch recipe is now part of the contribution system.** Develop and
  preserve planning artifacts on `dev`; cut public PR branches from `upstream/main`;
  bring over only intended product files; keep PR-body staging docs fork-local.

## Changes Made

| Change | Detail |
|--------|--------|
| **README `update` deep-dive** | Added a `#### Automatic update commands` subsection: `bash -c` in the collection dir, run-then-reindex order, non-zero-exit aborts the whole run, `qmd collection update-cmd` set/clear, sample `qmd update` trace. (`README.md`) |
| **README `ignore` correctness** | Now states YAML-only (no CLI) + additive with the un-overridable built-ins (`node_modules,.git,.cache,vendor,dist,build`); links the section to `example-index.yml`. |
| **`example-index.yml` overhaul** | 3 near-identical collections → 6, each demonstrating one feature (hierarchical context, auto-`update`, `ignore`, non-md globs, `includeByDefault:false`, all-fields); commented `editor_uri`/`models`. YAML-validated. |
| **CHANGELOG** | Two `[Unreleased] → Documentation` bullets for the above (committed `4e86ba0`). |
| **PR #718 body** | Rewrote `docs/pr-body-phase-1.25-index-yml.md` off the stale March layout to match what actually shipped; refreshed line numbers in the local-only HTML comment; stripped that comment at file time before posting. |
| **CLAUDE.md refresh** | Fixed `--pull`; added `init`/`doctor`/`bench`; added collection `show`/`update-cmd`/`include`/`exclude`; led output with `--format <kind>` (booleans = legacy aliases); documented `get <file>[:from[:count]]` + `--intent`/`--no-rerank`/`--full-path`/`--no-line-numbers`. (commit `7492ec4` on PR branch) |
| **Cloud routine** | Created `qmd-watch-716-718-719` (`trig_01WbeXf222N9L9RADtsEBW4p`), every 3h, GitHub public API via curl, mobile push on maintainer activity. |
| **Project memory** | Added `project_pr_watch_routine.md` + MEMORY.md pointer so the routine is discoverable next session. |

## Research / Verification Performed

- **Confirmed #715 merged** as a true merge commit (`6366024`, preserving `7488fe8`
  verbatim) — so upstream's Phase 1 content is byte-identical to ours, making the
  cherry-pick-onto-upstream/main approach produce clean, overlap-free diffs.
- **Re-verified every index.yml line citation** against current `dev` before filing:
  `collections.ts` schema `:27-34`, config-dir precedence `:114-121`, global `.yml`
  `:125`, project-local `.yaml`/`.yml` precedence `:138-143`; `qmd init` models seed
  `cli/qmd.ts:421`; `update` exec `cli/qmd.ts:683-717`; `ignore` merge
  `store.ts:1284-1296`. Only line *numbers* drifted; behavior intact.
- **Verified every CLAUDE.md command/flag against `src/cli/qmd.ts`** — caught two
  would-be errors from memory: `-C` is `--candidate-limit` (not `--full-path`), and
  `--intent`/`--format` are real CLI flags. Confirmed `--pull` still dead
  (`qmd.ts:2868` parsed, never consumed). Left `vector-search`/`deep-search` out
  (source labels them "undocumented alias").
- **Verified both PRs post-creation**: #718 = 3 files / 200+ /10-; #719 = 1 file /
  25+ /7-; both base `main`, bodies clean of leaked staging comments.

## Summary Statistics

- **2 PRs filed** (#718, #719); **1 confirmed merged** (#715)
- **4 product commits** on `dev` (`4e86ba0` index.yml strengthening, `22d5709` +
  `88b71e0` PR-body artifacts, plus the CLAUDE.md work) + `7492ec4` on the #719 branch
- **2 source files** newly verified for the CLAUDE.md PR; **~7** total audited across
  the session
- **1 cloud routine** created + **1 mobile push** verified
- **2 memory files** touched (1 new)

## Discoveries / Handoff Notes — Cloud Agents, Remote Control & PushNotification

> The user explicitly wants this preserved as the seed for a **future custom skill**
> around scheduled cloud monitoring + phone alerts. Capturing the mechanics in full.

**What a "cloud cron agent" is.** Two parts: *cron* (a 5-field UTC schedule, e.g.
`0 */3 * * *` = every 3h) + *cloud agent* (each firing boots a fully isolated,
headless Claude Code session in Anthropic's cloud — its own sandbox, a **fresh git
clone** of the named repo, only the `allowed_tools` you grant, running a
self-contained prompt). Key consequences: (a) runs whether or not the laptop is open;
(b) **zero memory of prior runs** and **no access to the local machine** — so prompts
must be fully self-contained and must NOT rely on local files or `gh` auth. Our
routine uses the **public GitHub REST API via curl** precisely so it can't silently
fail on missing credentials.

**Managing routines** — via the `RemoteTrigger` tool (load with
`ToolSearch select:RemoteTrigger`; OAuth handled in-process, never use curl for it):
`list` / `get` / `create` / `update` (partial — but pass the whole `job_config` to
change the prompt) / `run`. **You cannot delete** via the API — only at
https://claude.ai/code/routines/{id}. The `/schedule` skill wraps all of this and
also surfaces the available `environment_id` and connected MCP connectors. Minimum
cron interval is **1 hour**. Create body needs a fresh lowercase v4 UUID in
`events[].data.uuid`.

**How the "ping" reaches the phone — `PushNotification` + Remote Control.** The
`PushNotification` tool "sends a desktop notification in the user's terminal. **If
Remote Control is connected, it also pushes to their phone.**" So the path to the
Claude **iOS** app is: app installed + signed into the same account + **Remote
Control connected** + iOS notifications enabled. Add `PushNotification` to the
routine's `allowed_tools` and instruct it to fire **once, conditionally** (only on
real activity) with `status:"proactive"` and a one-line message <200 chars.

**The focus-suppression gotcha (important and non-obvious).** In an *interactive*
session, `PushNotification` is **suppressed when the terminal has focus** — it
returned `"Not sent — terminal has focus. Terminal + mobile suppressed."` This is by
design (you're obviously present). It is NOT evidence Remote Control is broken. The
**cloud routine has no terminal**, so this suppression does not apply to it — which
actually *improves* the odds the routine's push lands. When the user backgrounded the
terminal, the same test push **did arrive on the iPhone** (screenshot confirmed),
proving the app/Remote-Control side works.

**For the future skill.** Likely shape: a parameterized "watch these GH items, push
me on change" routine generator — inputs = repo + item numbers + interval; behavior =
public-API poll, conditional single push, dashboard report as backstop; teardown
reminder when items resolve. Consider: persisting last-seen state to avoid re-pushing
the same event (current design is stateless and would re-alert each run after the
first activity — acceptable here because the user engages and disables once tobi
acts, but a real skill should dedupe). Also consider routing to a messaging connector
(Slack) for users without Remote Control.

## Contribution Posture After This Session

The contribution sequence is now active rather than theoretical:

1. **#715 validated the lane** — documentation/discoverability improvements can merge
   upstream quickly when they are source-true and scoped.
2. **#718 is the substantive next proof point** — it is larger than #719, but it is
   anchored to an explicit maintainer-recognized gap: automatic `index.yml` update
   hooks and the broader declarative config surface.
3. **#719 is the small correctness probe** — if it merges quickly, that reinforces
   the "small stale-doc fix" lane; if it gets comments, use that feedback to calibrate
   future agent-facing docs.
4. **Phase 1.75 is ready but should be paced** — HTTP REST/ops docs are independent,
   but two open docs PRs plus one open issue is enough queue pressure for now.
5. **#716 remains code, not docs** — hold until maintainer signal or until the docs
   queue has cleared. The #699 lesson still applies: docs are not a substitute for
   a behavior fix when users remain functionally stranded.

## Current State

- On branch `dev`, working tree clean. Local staging artifacts committed to `dev`:
  `docs/pr-body-phase-1.25-index-yml.md`, `docs/pr-body-claude-md-refresh.md`
  (fork-only — never PR'd).
- Two PR branches pushed to `origin`: `docs/qmd-index-yml-config` (#718),
  `docs/qmd-claude-md-refresh` (#719).
- Cloud routine **live**: `qmd-watch-716-718-719`, next run ~02:01 AM PDT 2026-06-09.

## Unfinished Work / Next Steps

1. **Await #718 / #719 review** — routine will push on activity; dashboard backstop.
2. **Phase 1.75** (HTTP REST/ops docs, committed `da03b9e`) — fileable once #718/#719
   land; same cherry-pick-off-upstream/main recipe.
3. **#716 code** — hold for maintainer signal (no engagement yet).
4. **Phase 1.5** (library-first README reframe) — still held, reactive-only.
5. **Disable the routine** once #716/#718/#719 resolve (web UI to delete).
6. **Possible future work**: turn the cloud-watcher pattern into a reusable custom
   skill (see Discoveries).

## Issues & PRs

- **PR #718** — docs: document the index.yml configuration schema —
  https://github.com/tobi/qmd/pull/718 (OPEN)
- **PR #719** — docs: refresh stale command reference in CLAUDE.md —
  https://github.com/tobi/qmd/pull/719 (OPEN)
- **PR #715** — CLI/MCP/bench reference — https://github.com/tobi/qmd/pull/715 (MERGED)
- **Issue #716** — Per-subcommand `--help` + bench guardrails —
  https://github.com/tobi/qmd/issues/716 (OPEN, no signal)
- **Routine** — https://claude.ai/code/routines/trig_01WbeXf222N9L9RADtsEBW4p
