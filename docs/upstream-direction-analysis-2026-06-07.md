---
date: 2026-06-07
title: "Upstream tobi/qmd — Strategic Direction Analysis"
project: qmd
branch: dev
tags: [upstream, strategy, maintainer-intent, contribution-strategy, direction, tobi]
related_issues: [715, 716, 620, 699, 560, 649, 481, 631, 587, 470, 449]
related_pr: 715
summary: >
  Consolidated assessment of the upstream tobi/qmd project's intended direction,
  what gets merged vs dismissed, the maintainer's operating model, the community
  submission landscape (Apr–Jun 2026), and where our PR #715 / issue #716 and
  follow-on plans sit. Two independent analyses reconciled; all load-bearing
  citations verified against the GitHub API.
---

# Upstream `tobi/qmd` — Strategic Direction Analysis

**Date:** 2026-06-07 · **Repo:** [github.com/tobi/qmd](https://github.com/tobi/qmd)
(26,234 ⭐ / 1,646 forks, created Dec 2025 — a ~6-month-old project that went viral)
**Window analyzed:** CHANGELOG arc (v0.1.0 → Unreleased) + all 92 issues and ~120 PRs
from Apr 1 – Jun 7, 2026, plus @tobi's directional comments.
**Method:** Two independent analyses reconciled. Every load-bearing citation below
was verified verbatim against the GitHub API.

---

## TL;DR

We are **not outliers**. Our #715/#716 direction sits squarely in the project's
dominant trajectory: QMD as a **local-first document-retrieval layer for agents**,
with heavy current investment in CLI/MCP/HTTP ergonomics, reproducible search
quality, install/runtime reliability, and model/config clarity.

The merge bar is: **small, correct, local-ethos, reviewable.** The rejection bar is:
**anything that expands QMD into an adjacent product or forks the single runtime
story.** Our chosen lane — docs/discoverability/UX correctness — is *underserved*
and mirrors the maintainer's own current priorities.

---

## 1. The maintainer's operating model

This is a **maintainer-as-author** project, not maintainer-as-curator. That shapes
everything.

| Pattern | Evidence |
|---------|----------|
| **Batch re-implementation** | ~12 community PRs fixing "model config ignored / hardcoded defaults" (#525, #527, #531, #559, #564, #571, #625, #638…) were **all closed unmerged** with an identical note: *"superseded by the v2.5.x release line… covered by the centralized model/GPU/config fixes."* tobi took the *reported problem* and shipped his own consolidated fix ([#636](https://github.com/tobi/qmd/pull/636), +1,200). |
| **Umbrella consolidation** | Remote-backend requests (#403, #428, #489, #521) were closed as duplicates and funneled into one open design issue, [#620](https://github.com/tobi/qmd/issues/620). He refuses point solutions in favor of one coherent design. |
| **Stale-closing** | [#516](https://github.com/tobi/qmd/issues/516) "Is this project dead?" → *"The project is active. Closing as stale/no longer useful for tracking work."* |
| **Direct-merges small, targeted fixes** | 58 of 82 recently-merged community PRs were ≤60 lines. The bar for a small, correct, local-ethos fix is low and fast. |

**Tactical implication:** the batch-and-supersede habit is a real risk for our
*code* PR (#716). Community contributors did genuine work and got closed as
"superseded." Mitigate with small scope, tests, and convention citations so the
change is cheaper to merge than to re-implement.

---

## 2. What gets merged

- **Small bug fixes matching existing architecture** — the bread and butter
  (e.g. #644 absolute line numbers, #635 ls abs paths, #555 Windows HOME one-liner,
  and our own [#533](https://github.com/tobi/qmd/pull/533)).
- **Big features only when they (a) extend local retrieval quality and (b) arrived
  before the current consolidation phase.** The large community features that *did*
  land — [#449](https://github.com/tobi/qmd/pull/449) AST chunking (+1,910),
  [#470](https://github.com/tobi/qmd/pull/470) `qmd bench` (+612), #395 bounded embed
  memory — all merged in **March** and all reinforce the local-inference core.
- **Diagnostics & onboarding-correctness is the current tobi priority** — he's
  authoring these himself: #659 `qmd doctor`, #656 GPU/bench warnings, #657 serve
  skill from CLI, #655 project-local indexes.

**Maintainer enthusiasm (verified quotes):**
- [#470](https://github.com/tobi/qmd/pull/470#issuecomment-4189517991) `qmd bench`: *"yea that's a good idea"*
- [#449](https://github.com/tobi/qmd/pull/449#issuecomment-4149062985) AST chunking: *"Great contribution — the AST-aware chunking is exactly what QMD needed for code search."*

---

## 3. What gets dismissed — the DO-NOT list

Rejections cluster on one theme: **expanding QMD into an adjacent product, or
forking the single local-runtime story.** Verified maintainer quotes:

| Don't propose | Issue | tobi's verbatim reason |
|---------------|-------|------------------------|
| **Remote/cloud backends as point solutions** | ~10 PRs (#480, #490, #503, #509, #517, #575, #603, #619, #629, #705) | All closed-unmerged or untouched. *Only* sanctioned path is [#620](https://github.com/tobi/qmd/issues/620) (one OpenAI-compatible design, with tests). |
| **Alternative model runtimes (MLX)** | [#649](https://github.com/tobi/qmd/issues/649#issuecomment-4468604359) | *"Not planning an MLX backend. QMD's local inference stack is intentionally built around GGUF/node-llama-cpp so install/runtime behavior stays one path across platforms."* |
| **Parallel SDKs (Python)** | [#560](https://github.com/tobi/qmd/issues/560#issuecomment-4468604731) | *"a second SDK would add maintenance cost and drift from the core TypeScript implementation."* |
| **Knowledge graph** | [#481](https://github.com/tobi/qmd/issues/481#issuecomment-4468606310) | *"a different product surface and a lot of complexity."* |
| **Ad-hoc text comparison** | [#631](https://github.com/tobi/qmd/issues/631#issuecomment-4468604460) | *"turns QMD into a general reranker/similarity utility instead of a… search/retrieval tool."* |
| **Write-side MCP tools (`update`/`embed`)** | [#587](https://github.com/tobi/qmd/issues/587#issuecomment-4468604645) | *"The MCP surface is intentionally read-oriented; exposing `update`/`embed` over a daemon makes it too easy for clients to trigger expensive or destructive index work remotely."* |
| **External README badges** | [#610](https://github.com/tobi/qmd/pull/610#issuecomment-4365013876) | Closed. |
| **Feature expansions beyond the core** | #484 symbol-extraction "Phase 2" (closed though Phase 1 merged), #522 section-level filtering, galligan's chunking scanner suite #538–541 | — |

---

## 4. The docs nuance that sharpens our whole plan (#699)

The most decision-relevant single finding. On
[#699](https://github.com/tobi/qmd/issues/699#issuecomment-4638716651) (npm install
fails on Apple Silicon), tobi posted a full decision rejecting **docs-only** as a fix:

> *"(c) Docs-only: Leaves users stranded with a bare `npm error code 1` and zero
> path forward without reading docs. Not good enough."*

He chose to make `node-llama-cpp` an optional dependency with graceful degradation,
and added he'd build doctor arch-detection *"as a bonus."*

**Implication:** docs are welcome as **clarification of shipped behavior**, but never
as a **substitute for a code fix**. This validates that #716 (the *code* companion to
#715's docs) is **necessary, not optional** — and is a standing caution against any
"just document around it" instinct.

---

## 5. Community submission landscape (Apr–Jun 2026)

92 issues / ~120 PRs. The community uses QMD exactly as we do: agent-facing local
retrieval over notes/code/docs, often via MCP, with pain around install, GPU/CPU
runtime, model config, path fidelity, and search quality.

| Cluster | Volume | Maintainer disposition |
|---------|--------|------------------------|
| Remote/OpenAI-compatible backends (#517, #619, #629, #689, #705) | Largest | Funneled to #620; otherwise rejected |
| Model/config clarity via `index.yml` (#678, #645, #502) | Very high | Fixed by tobi's batch; PRs superseded |
| Agent/MCP ergonomics (#648, #644, #606) | High | Small fixes merged readily |
| Search quality (#685, #697, #704, #703) | Medium | Mixed |
| Cross-platform packaging (Windows, Apple Silicon, Bun/Node, SQLite, Metal, CUDA/Vulkan) | High | Largely unaddressed — accumulating debt |
| **Docs / discoverability / UX** | **Low** | **Underserved — our lane** |

Note on counts: a strict 2-month-windowed pass tallied **117 PRs: 27 merged, 54 open,
36 closed unmerged** (16 of the merged external, 11 maintainer-authored). Exact splits
depend on the window; the *ratios* — external welcomed, many open/superseded — are
robust across both analyses.

---

## 6. Where we stand — PR #715 / issue #716

| Factor | Our work | Fit |
|--------|----------|-----|
| **Track record** | [#533](https://github.com/tobi/qmd/pull/533) merged Apr 9; #385 (+112) merged earlier | ✅ Proven small-fix merger |
| **[#715](https://github.com/tobi/qmd/pull/715)** = pure docs + 1-word MCP fix | Documents exactly what upstream has been changing (CLI flags, MCP params, bench, doctor, collection semantics, `get` ranges, formats). No behavior change. | ✅ Strong fit; main risk is **size, not direction** |
| **[#716](https://github.com/tobi/qmd/issues/716)** = per-command `--help` + bench guardrails | Code change, local-only, no runtime fork, improves correctness/UX | ✅ Fits; manage batch-supersede risk |
| **We avoid every rejected category** | Zero remote backends, runtimes, SDKs, graph, write-MCP | ✅ Cleanly outside all rejection buckets |

---

## 7. Committed contribution sequence

1. **Phase 1 — #715** (in flight) → await maintainer signal. Be ready to split/trim
   if review friction appears; substance is aligned.
2. **Phase 1.25 — `index.yml` config docs PR** — next move. Concrete,
   source-verifiable, independently demanded (#678, #645). Highest-fit, lowest-objection.
3. **Phase 2 — #716 code** — framed as a *"discoverability guardrail for
   already-shipped commands,"* **not** a new UX system. Shape it boring:
   - No new help framework unless the existing CLI parser forces it.
   - Tests for `qmd <subcommand> --help` as **read-only assertions over current
     output** (a test that demands flag *registration* is a framework in disguise —
     avoid).
   - Compact output; no broader CLI rewrite.
   - PR body leads with the regression-style problem (*"every `--help` emits the same
     97-line block"*) before any proposal.
4. **Phase 1.5 — library-first README reframe — HELD, reactive only.** Positioning is
   taste/strategy, and a maintainer-as-author owns voice and framing. Don't lead with
   it; possibly never PR it cold — hold it as a response if tobi signals appetite.

> **Also in flight (not part of this analysis's sequence):** **Phase 1.75 — HTTP
> server REST/ops docs** (`docs/http-server-rest-api-and-management.md`, committed
> 2026-06-07). Independent of 1.25, ordered by priority not dependency, no gating.
> The authoritative roadmap for all phases is
> `docs/plan-documentation-sprint-2026-06-06-canonical.md`; this section reflects only
> the upstream-strategy view that motivated the 1.25-before-1.5 ordering.

---

## 8. Standing DO-NOT list (for all future proposals)

- ❌ No Python (or any second-language) SDK — #560
- ❌ No MLX or alternative model runtime — #649
- ❌ No knowledge-graph features — #481
- ❌ No ad-hoc text-compare command — #631
- ❌ No write-side MCP tools (`update`/`embed` over the daemon) — #587
- ❌ No remote/cloud backend except as a contribution to the **#620** umbrella
  (one OpenAI-compatible design, VCR-style tests)
- ❌ No docs-only fix where users remain functionally stranded — #699
- ⚠️ No unsolicited README/identity reframing — gate on maintainer signal
