# Documentation Accuracy Audit

**Date**: 2026-03-01
**Scope**: All session summary documents in `docs/` (Feb 24 - Mar 1) and `skills/qmd/` skill files
**Method**: Cross-referenced claims against the live system state (git, running processes, config files, source code)

## Why This Audit Exists

Six session summaries were written across Feb 24 - Mar 1 as the MCP multi-client feature was developed, debugged, and deployed. Each doc was accurate at the time it was written, but later sessions resolved issues without back-annotating earlier docs. This audit captures what is now outdated so future readers don't act on stale information.

---

## Session Summary Audit

### `session-summary-2026-02-24-gemini-agent-analysis.md`

| Line(s) | Outdated Claim | Current Reality |
|---------|---------------|-----------------|
| 19 | "global `qmd` v1.0.7 lacks a truncation fix, causing 2048+ token documents to trigger a native Metal crash" | Global is v1.1.0 (`~/.bun/bin/qmd`); crash fixed |
| 32 | Version discrepancy: "v1.0.7 (Buggy)" vs "v1.1.0 (Fixed)" | No discrepancy — global is v1.1.0 |
| 80 | Action: "Pin Zod to 4.2.1" | Done (commit `652fecc`) |
| 81 | Action: "Build & Link to promote local fixes to global" | Done (Mar 1 session) |
| 83 | Action: "Terminate the 8+ residual qmd mcp orphan processes" | Done (14 killed, Mar 1 session) |
| 84 | Action: "Revert marketplace.json to stdio" | Done (Feb 24 phase 2) |

**Still accurate**: Line 82 (inference lock/concurrency semaphore) — still an open enhancement.

---

### `session-summary-2026-02-24-gemini-mcp-multi-client.md`

This doc has the most outdated content — it was written mid-crisis before the build was fixed.

| Line(s) | Outdated Claim | Current Reality |
|---------|---------------|-----------------|
| 14 | "Used `npx tsx src/qmd.ts mcp --http` to avoid the broken `tsc` build" | Build is fixed; LaunchAgent runs compiled `dist/` via `~/.bun/bin/qmd` |
| 49-55 | **Entire "Build Broken" section** — "bun run build fails with ~12 errors", "Cannot compile TypeScript to dist/", "globally installed qmd CLI stays at v1.0.7" | Build works (zod pinned to 4.2.1). Global binary is v1.1.0 |
| 55 | "Workaround: Run from source with `npx tsx`" | No workaround needed |
| 57-61 | **Entire "Installed qmd v1.0.7" section** — crash description, fix path | Global is v1.1.0; no version gap |
| 73 | "MCP HTTP server: Running from source via `npx tsx` (background process)" | Running via LaunchAgent with compiled binary |
| 74 | "LaunchAgent: Stopped via `launchctl bootout`" | LaunchAgent is running |
| 75 | "Installed CLI: v1.0.7 at `~/.nvm/versions/node/v24.12.0/bin/qmd`" | Global is `~/.bun/bin/qmd` v1.1.0 |
| 80-88 | **Entire "How to Run" section** — recommends `npx tsx` everywhere, says LaunchAgent "NOT recommended" | Standard `qmd` works; LaunchAgent is the recommended run mode |
| 93-98 | "Fix Zod Build Compatibility (Blocking)" listed as unfinished | Done |
| 100-104 | "Publish v1.1.0 Release" | Linked locally; npm publish still deferred |
| 114-118 | "Update LaunchAgent to Run from Source" | Done — uses `~/.bun/bin/qmd` |

**Still accurate**: Session timeout concern (5 min, lines 11-12), M3 memory pressure (lines 63-69).

---

### `session-summary-2026-02-24-mcp-http-migration.md`

| Line(s) | Outdated Claim | Current Reality |
|---------|---------------|-----------------|
| 8 | "Reverted to nvm paths after Homebrew's Node 25 proved incompatible" | LaunchAgent now uses `~/.bun/bin/qmd`, not nvm |
| 18 | Decision: "Use nvm's Node 24 for LaunchAgent" | Superseded — LaunchAgent uses `~/.bun/bin/qmd` |
| 100 | "LaunchAgent: uses nvm Node 24 path" | Uses `~/.bun/bin/qmd` |
| 104 | "Git state: Detached HEAD at `5233e67`" | On `dev` branch |
| 108 | Unfinished: "Reattach HEAD to a branch" | Done |
| 110 | Unfinished: "Gemini agent setup" | Done |

**Still accurate**: The Homebrew detour narrative (lines 36-53) and marketplace.json saga (lines 66-93) are correct as historical accounts. Lessons learned (lines 112-122) remain valid.

---

### `session-summary-2026-02-25-observer-mcp-multi-client.md`

| Line(s) | Outdated Claim | Current Reality |
|---------|---------------|-----------------|
| 47-51 | Runtime table: `qmd` (installed) row says "**Crashes on long docs**" at "npm v1.0.7" | Global is v1.1.0; no crash |
| 55-56 | "LaunchAgent points to `~/.nvm/.../bin/qmd` (the installed v1.0.7). The primary agent bypassed this by running `npx tsx` in the foreground" | LaunchAgent points to `~/.bun/bin/qmd` (v1.1.0); no bypass needed |
| 102 | Blocking: "Fix Zod/tsc build" | Done |
| 103 | Blocking: "Publish v1.1.0" | Linked locally; npm publish deferred |
| 104 | Blocking: "Update LaunchAgent — currently stopped" | Done and running |

**Still accurate**: McpServer cleanup question (line 68), session timeout/concurrency items (lines 107-109), Bun vec0 limitation (lines 37-43), open questions (lines 119-123).

---

### `session-summary-2026-02-25-zod-fix-and-docs.md`

| Line(s) | Outdated Claim | Current Reality |
|---------|---------------|-----------------|
| 50 | "LaunchAgent: still stopped" | Running (PID active) |
| 50 | "LaunchAgent plist points to the nvm Node 24 path and the installed v1.0.7 binary" | Points to `~/.bun/bin/qmd` (v1.1.0) |
| 55 | Unfinished: "Update LaunchAgent" | Done |

**Still accurate**: npm publish deferred (line 54), session timeout (line 56), concurrency semaphore (line 57).

---

### `session-summary-2026-03-01-launchagent-v1.1.0-deploy.md`

Most recent doc; mostly accurate.

| Line(s) | Outdated Claim | Current Reality |
|---------|---------------|-----------------|
| 75 | "qmd 1.1.0 (fa3cf12)" | Now `qmd 1.1.0 (29209f7)` — a later commit was added after verification |
| 129 | Unfinished: "Commit the doc updates — CLAUDE.md, GEMINI.md, README.md are unstaged" | Done (committed in `29209f7`) |

**Still open**: `feature/mcp-multi-client` branch cleanup (line 132, branch still exists locally and on remote), session timeout (line 130), version in `/health` (line 131).

---

## Recurring Outdated Claims

Three claims appear across **multiple docs** and are wrong in all of them:

1. **"LaunchAgent uses nvm path / v1.0.7"** — docs 2, 3, 4, 5. Fixed Mar 1.
2. **"Build is broken / use `npx tsx` as workaround"** — docs 2, 4. Fixed by zod pin (doc 5).
3. **"Global qmd is v1.0.7"** — docs 1, 2, 4, 5. Updated to v1.1.0 on Mar 1.

---

## Skill File Audit: `skills/qmd/`

### `skills/qmd/SKILL.md`

| Line(s) | Issue | Detail |
|---------|-------|--------|
| 5 | Minor omission | `Install via npm install -g @tobilu/qmd` — doesn't mention `bun install -g` as alternative |
| 58-61 | **Documents a nonexistent query type** | References `expand` as a query type ("Use a single-line query (implicit) or `expand: question`"). The MCP `query` tool only accepts `lex`, `vec`, and `hyde` types. `expand` is a CLI-only concept from `qmd query` — it does not exist in the MCP tool schema. |
| 68 | Same `expand` reference | "Don't know vocabulary → Use a single-line query (implicit `expand:`)" — misleading for MCP consumers |

### `skills/qmd/references/mcp-setup.md`

| Line(s) | Issue | Detail |
|---------|-------|--------|
| 13-19 | Stale Claude Code config | Shows `~/.claude/settings.json` with stdio config. This still works as a fallback, but doesn't reflect the current HTTP setup via `~/.claude/.mcp.json`. Missing any mention of the HTTP connection method. |
| 52 | **Wrong tool name** | Says `structured_search` — the actual MCP tool is named `query` |
| 64 | Wrong parameter shape | `"collection": "optional"` (singular string) — should be `"collections": ["name"]` (plural, array) |

---

## Items Still Accurately Open

These appear as "unfinished" across multiple docs and are confirmed still unresolved:

| Item | Mentioned In |
|------|-------------|
| Bump session idle timeout from 5 min to 15-30 min | Docs 2, 4, 5, 6 |
| Add LLM concurrency semaphore for M3 memory pressure | Docs 1, 2, 4, 5 |
| Publish v1.1.0 to npm registry | Docs 2, 4, 5 |
| Add version to `/health` response | Doc 6 |
| Delete `feature/mcp-multi-client` branch | Doc 6 |
| Verify `McpServer.close()` needed alongside `transport.close()` on session expiry | Doc 4 |
| Document Bun's sqlite-vec limitation in CLAUDE.md | Doc 4 |
