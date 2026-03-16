# Session Summary: Phase 5 — CLAUDE.md / CLAUDE.local.md Rebuild

**Date:** 2026-03-16
**Branch:** `dev`
**QMD version:** v2.0.1 (source build)

---

## Purpose

Complete Phase 5 of the v2.0 upgrade plan: rebuild the CLAUDE.md / CLAUDE.local.md pair to eliminate stale instructions that were actively misleading session agents. This session was triggered by three consecutive errors caused by the incomplete Phase 5:

1. Agent recommended `bun install` (CLAUDE.md said "Use Bun")
2. Agent recommended `qmd mcp stop` (CLAUDE.md listed it as a command, but it doesn't work with our LaunchAgent setup)
3. Agent had no awareness of the LaunchAgent at all

---

## What Went Wrong (Root Cause)

During the v2.0 upgrade (2026-03-11), CLAUDE.local.md was renamed to `.bak` to prevent stale v1.1.0 instructions from loading. Phase 5 (reconcile the two files) was deferred. Five days later, every new session agent was reading only the upstream CLAUDE.md — which contains instructions that are **wrong for our local environment**:

- "Use Bun instead of Node.js" — we switched to Node/npm because Bun breaks sqlite-vec
- `bun install`, `bun link`, `bun test` — all wrong; we use npm exclusively
- `qmd mcp stop` — doesn't work; our daemon is managed by a macOS LaunchAgent
- No mention of the LaunchAgent, the three-layer integration architecture, or the fact that we're running from source (not the published npm package)

---

## Changes Made

| Change | Detail |
|--------|--------|
| **CLAUDE.md reverted to upstream** | `git checkout main -- CLAUDE.md` — no local modifications, so no merge conflicts on future `git pull` from upstream |
| **CLAUDE.local.md written from scratch** | Comprehensive local environment file replacing the stale `.bak`. Contains override language, install state, runtime rules, LaunchAgent management, three-layer architecture, v2.0 source layout, database schema, search pipeline |
| **Deleted stale backups** | Removed `CLAUDE.local.md.bak` (old v1.1.0 instructions) and `CLAUDE-temp.md.bak` (pre-v2 reference copy) |
| **Memory: feedback_no_bun.md** | Created — never use Bun, CLAUDE.local.md overrides CLAUDE.md |
| **Memory: feedback_no_editorializing.md** | Created — don't inject precautionary steps that contradict facts |
| **Memory: reference_launchagent.md** | Created — LaunchAgent management details, stop/start commands |
| **Memory: MEMORY.md** | Updated with new Feedback and References sections |

---

## Key Design Decision: Two-File Strategy

**CLAUDE.md** (tracked, shared) = upstream's general project instructions. We do NOT modify it. This prevents merge conflicts on every `git pull` from `tobi/qmd`.

**CLAUDE.local.md** (gitignored via `*.md` rule) = our local environment truth. Has explicit "OVERRIDE" sections that call out where CLAUDE.md is wrong for us. The preamble states in bold: **"Where this file contradicts CLAUDE.md, THIS FILE WINS."**

This separation means:
- Pulling upstream never clobbers our local instructions
- Our local instructions never accidentally get pushed upstream
- Session agents see both files and know which one takes precedence

---

## CLAUDE.local.md Contents (for reference)

The new file covers:

1. **Current Install State** — from source, not npm. Binary path, symlink chain, version. Has a callout to update this section whenever it changes.
2. **Update/Reinstall Workflow** — `launchctl unload` → `git pull` → `npm run build && npm link` → `launchctl load`
3. **Runtime Override** — Node.js, NOT Bun. Explains why (sqlite-vec loadExtension), lists rules (never bun install, never bun test, etc.), explicitly calls out CLAUDE.md's Bun instructions as wrong.
4. **MCP Server Override** — LaunchAgent managed, not `qmd mcp --daemon`. Plist path, label, logs. Explains why `qmd mcp stop` and `kill` don't work. Correct stop/start commands.
5. **Three-Layer Architecture** — MCP connection (tools), Plugin (skills/knowledge), `qmd skill install` (redundant for us).
6. **v2.0 Architecture** — Source layout (`src/cli/`, `src/mcp/`, `src/index.ts` SDK layer), dependency flow.
7. **Database Schema** — All 6 tables with purposes.
8. **Search Pipeline** — 7-step hybrid query pipeline.
9. **Agent Preferences** — subagent model: sonnet, not haiku.

---

## Phase Status Update

| Phase | Status | Notes |
|-------|--------|-------|
| 0: Safety Tags | Done | |
| 1: Update main | Done | |
| 2: Assess dev | Done | |
| 3: Rebuild dev | Done | |
| 4a: Stop LaunchAgent | Done | |
| 4b: Build and install | Done | |
| 4c: Assess MCP + LaunchAgent | Done | |
| **5: CLAUDE file rebuild** | **Done (this session)** | |
| 6: Idle Timeout PR | Implemented on dev, not submitted upstream | Code + 7 tests exist; needs own branch + PR |

**The v2.0 upgrade phased plan is now complete through Phase 5.** Only Phase 6 (optional upstream PR for session idle timeout) remains.

---

## Handoff Notes

### What the next session agent should know

- **CLAUDE.local.md is the authority.** If CLAUDE.md says something different (especially about Bun, daemon management, or development commands), CLAUDE.local.md wins. The file says this explicitly at the top.
- **CLAUDE.md is untouched upstream.** It will say "Use Bun" — that's upstream's recommendation, not ours. Don't follow it.
- **The update workflow is documented.** "Update/Reinstall from Source" section in CLAUDE.local.md has the exact 4 commands. Don't improvise alternatives.

### Outstanding work on dev branch (uncommitted)

- `src/mcp/server.ts` — session idle timeout (30 min) + configurable sweep interval
- `test/mcp.test.ts` — 7 new session lifecycle tests
- `.claude-plugin/marketplace.json` — mcpServers block removed (skills-only plugin)

### Outstanding upstream work

- **PR #384** (hyphen fix) — open, no reviews after 4+ days
- **Session timeout PR** — Phase 6, code exists on dev but needs its own branch off main
- **PR candidates A/B/C** from `docs/pr-candidates-2026-03-12.md` — unstarted (marketplace.json cleanup, skill docs, README plugin section)

---

## Summary Statistics

- 2 CLAUDE files rebuilt (1 reverted, 1 written from scratch)
- 2 stale backup files deleted
- 3 memory files created
- 1 memory file updated
- 0 code changes
- Phase 5 of 6 completed in the v2.0 upgrade plan
