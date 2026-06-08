---
session_id: 73c48733-6ad4-4e1b-bcdc-fe0eab2e710b
date: 2026-06-07
time: "4:38 PM PDT – 5:36 PM PDT"
project: qmd
branch: dev
---

# Session Summary: `dev-old` Branch Salvage & Retirement

## Overview

Investigated the stale `dev-old` branch to confirm it held no lost
documentation-sprint Phase 1 work, salvaged the one piece of genuinely unique
markdown content it contained (HTTP server REST API + ops docs), verified that
content against current v2.5.3 source, recorded it as a future docs-sprint
candidate (Phase 1.75), and retired the branch.

## How We Got Here

A note in the 2026-06-07 doc-sprint Phase 1 session summary ("Phase 1 PR not yet
submitted… edits exist locally on dev") prompted the user to wonder whether
work-in-progress was quietly sitting in the long-lived `dev-old` branch. It wasn't
clear when `dev-old` was created or what it held. This session set out to inspect
it and assess whether anything intended for Phase 1 had been missed.

## Key Decisions Made

| Decision | Rationale |
|----------|-----------|
| **`dev-old` holds no Phase 1 work** | Its newest commit (`f5597ba`, 2026-03-11) predates the canonical Phase 1 plan (2026-03-17) entirely. Phase 1 was never committed anywhere until this week — it could not have been in `dev-old`. |
| **Salvage only the README ops content** | 9 of 12 `dev-old`-authored docs are byte-identical in `dev` (migration carried them over); `branch-assessment`/`CLAUDE.md` are superseded. Only the README HTTP/REST sections were unique and unrecovered. |
| **Standalone artifact, not folded into a phase** | Kept the salvage as `docs/http-server-rest-api-and-management.md` and referenced it from the plan rather than forcing it into Phase 1.25 (which is cleanly about `index.yml`). |
| **Defer to a new Phase 1.75** | HTTP server operations is a different review shape (runtime ops manual) than Phase 1's CLI/MCP reference. Deferral preserves PR reviewability. |
| **Phase 1.75 is priority-ordered, not dependency-gated** | Its source is verified and self-contained; it can proceed anytime, independent of 1.25/2's maintainer-signal gates. |
| **Retire `dev-old`** | Once fully catalogued and drained, no reason to keep it. |

## Changes Made

| Change | Detail |
|--------|--------|
| **Salvaged HTTP server doc** | Created `docs/http-server-rest-api-and-management.md` (187 lines) — REST `/query`/`/search` reference + MCP HTTP server management (daemon vs LaunchAgent, restart/health/orphan-kill, troubleshooting, graceful shutdown, session management), extracted from `dev-old` README @ `f5597ba`. |
| **Corrected two drifted claims** | (1) 5-min idle timer disposes models too (`disposeModelsOnInactivity: true` in `src/index.ts`), not just contexts; (2) there is **no** idle-session timeout — sessions live until transport close/restart. Both flagged inline with `[Corrected 2026-06-07]`. |
| **User precision edits (verified)** | `/mcp` documented as POST/GET/DELETE; request table gained `candidateLimit`/`intent`/`rerank`; `collections` default clarified (default-included → fallback all); response shape gained `line`; `qmd status` scoped to daemon/PID-file. All confirmed against `src/mcp/server.ts`. |
| **Canonical plan: Phase 1.75 note** | Added to `docs/plan-documentation-sprint-2026-06-06-canonical.md` — candidate framing, salvage pointer, "keep separate from 1.25," canonical PR thesis sentence, and a "Gating: none" clarification. |
| **Deleted `dev-old`** | `git branch -D dev-old` (was `f5597ba`). No remote counterpart existed. |

## Research / Verification Performed

- **Branch topology**: `dev-old` merge-base `d6f3688`, 9 unique commits (all
  2026-02-24 → 2026-03-11 MCP multi-client / HTTP-migration work), "ahead 9,
  behind 285" of `origin/dev`. Touches `src/mcp.ts` (pre-v2.0 layout) — predates
  the v2.0 split.
- **Markdown inventory**: compared all 27 `dev-old` `.md` files against `dev`'s 59.
  No path is exclusive to `dev-old`; content comparison showed 9 authored docs
  identical in `dev`, 3 differing (2 superseded, 1 = the unique README content).
- **Feature-supersession check**: `dev-old`'s headline `feat(mcp)` multi-client
  commit (`756c89e`) is **not** an ancestor of `dev`/`origin/main`, yet current
  `src/mcp/server.ts` already implements multi-client sessions natively — upstream
  re-implemented it in the v2.0 rewrite.
- **Source verification of salvaged claims** against `dev`:
  `/health`/`/query`/`/search`/`/mcp` endpoints exist (`server.ts:684/694/749/796`);
  `/query` reads `candidateLimit`/`intent`/`rerank`; `/mcp` non-POST handler
  serves GET/DELETE with session validation; `DEFAULT_INACTIVITY_TIMEOUT_MS =
  5*60*1000` with `disposeModelsOnInactivity: true` (`index.ts:377-378`); zero
  session-expiry timers in `server.ts`.
- **Remote check**: no `origin/dev-old`; local `dev-old` tracked `origin/dev`
  (rename fingerprint). Fork has 6 branches; `feature/mcp-multi-client` still
  present (the source of the multi-client work — a future retirement candidate).

## Summary Statistics

- **2 product/doc files** changed (+200 insertions): 1 new salvage doc (187 ln),
  1 plan note (+13 ln initial, plus this session's gating/thesis additions).
- **27 markdown files** inventoried across `dev-old` vs `dev`.
- **1 branch deleted** (`dev-old`); 0 remote refs affected.
- **2 drifted doc claims** corrected; **6 user precision edits** source-verified.

## Current State

- Checked-out branch: `dev`.
- Staged/modified for the intended commit:
  `docs/http-server-rest-api-and-management.md` (new),
  `docs/plan-documentation-sprint-2026-06-06-canonical.md` (Phase 1.75 note),
  and this session summary.
- `dev-old` no longer exists locally; no remote counterpart ever existed.
- **Not part of this work** (pre-existing untracked, keep out of the commit):
  `docs/replay-7851355b.md`, `docs/upstream-direction-analysis-2026-06-07.md`.

## Discoveries / Handoff Notes

- **`dev-old` was the pre-v2.0 `dev`.** `branch.dev-old.merge = refs/heads/dev` is
  the fingerprint of a `git branch -m dev dev-old` rename — exactly what the v2.0
  migration plan (`branch-assessment-2026-03-11`) prescribed. It was set aside
  intact during the v2.0 cutover, not abandoned mid-work.
- **The v2.0 migration was clean.** Every local project doc (`node-101`, 7 session
  summaries, `GEMINI.md`) was carried into `dev` byte-identically. The README was
  the one category deliberately *not* copied ("upstream rewritten"), which is why
  its ops sections were the only orphaned content.
- **Canonical PR thesis for Phase 1.75** (worth not re-deriving): "QMD has an
  undocumented HTTP server surface: REST `/query`/`/search` plus operational
  guidance for daemon and LaunchAgent modes."
- **Future split decision for Phase 1.75**: the REST `/query`/`/search` reference
  is clean upstream-general material; the LaunchAgent/orphaned-process ops content
  is macOS-specific and overlaps local `CLAUDE.local.md` territory. When the phase
  is picked up, decide whether to send only the REST reference upstream and keep
  the ops specifics local.

## Unfinished Work / Next Steps

1. **Commit** the salvage doc + plan note + this summary together (commit message
   provided separately in-session).
2. **Phase 1.75** remains staged — pick up when there's maintainer appetite for
   HTTP-transport docs; first decision is the upstream/local split above.
3. **Optional**: `feature/mcp-multi-client` (local + on `origin`) is a retirement
   candidate — its feature was superseded by upstream's v2.0 multi-client
   implementation. Not assessed in depth this session.
