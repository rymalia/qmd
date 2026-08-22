---
session_id: 3960cc00-9dd5-4114-94d7-27def60c0356
date: 2026-08-21
time: "6:09 PM PDT – 6:19 PM PDT"
project: qmd
branch: dev
---

# Session Summary — Switch qmd install from local source to published npm

## Overview

Switched the local `qmd` install from source-mode (`npm link` → `/Users/rymalia/projects/qmd`, running `tsx src/cli/qmd.ts`) to the **published `@tobilu/qmd@2.8.3`** npm package, restarted the LaunchAgent-managed MCP daemon onto the compiled build, and verified the full stack (doctor, live query, MCP HTTP endpoint) against the existing index. Updated the authoritative `CLAUDE.local.md` install-state section to match.

## Key Decisions Made

| Decision | Rationale |
|----------|-----------|
| **Proceed with the switch despite it being an upgrade, not a downgrade** | The doc's note that "latest published is v2.5.3" was stale; `npm view` showed `latest` = `2.8.3`. Moving 2.6.3 (source) → 2.8.3 (published) is newer code, and the switch is trivially reversible (`npm link` from the repo restores source mode), so there was no reason to block. |
| **Re-run install with `--allow-scripts` for the native packages** | The first `npm install -g` completed but npm's default script-blocking silently blocked `node-llama-cpp`'s postinstall (downloads the llama.cpp binary for embeddings/rerank/query-expansion) and the tree-sitter `node-gyp-build` steps. Without them the LLM and AST-chunking layers would fail at runtime. Used a scoped one-time `--allow-scripts` (npm's own suggested fix) rather than changing the global block config. |
| **Leave the plugin layer (`known_marketplaces.json`) pointing at the local repo** | The user asked to switch the *install* (binary/package) only. The plugin provides SKILL.md knowledge and is independent of the binary; changing it was out of scope. Flagged as an optional follow-up. |
| **Restart the daemon via `launchctl unload`/`load`, no plist edit** | The plist's `ProgramArguments[0]` is the stable path `…/v24.18.0/bin/qmd`, which the global install reuses, so the path needed no change. |

## Changes Made

| Change | Detail |
|--------|--------|
| **Removed the npm link** | `npm rm -g @tobilu/qmd` — removed the global symlink into `projects/qmd`; source repo untouched. |
| **Installed published package** | `npm install -g --allow-scripts=node-llama-cpp,tree-sitter-go,tree-sitter-python,tree-sitter-rust,tree-sitter-typescript,tree-sitter-javascript @tobilu/qmd` → `@tobilu/qmd@2.8.3`. Binary now resolves to a real file at `~/.nvm/.../lib/node_modules/@tobilu/qmd/bin/qmd`. |
| **Restarted the MCP daemon** | `launchctl unload` before the swap, `launchctl load` after. New process (PID 65982/65987, started 18:15:24) runs `node dist/cli/qmd.js mcp --http` — no tsx/src. |
| **Updated `CLAUDE.local.md`** | Rewrote the TL;DR and "Current Install State" section: install method (published npm), version 2.8.3, real-file symlink chain, dist-via-node runtime, the required `--allow-scripts` install command, the source-repo-decoupled note, a documented revert-to-source path, and a note that the source-mode subsections no longer apply while on the published install. Uncommitted (staged for the user to commit). |

## Testing / Research Performed

- `npm view @tobilu/qmd version` / `dist-tags` → confirmed published `latest` = `2.8.3` (contradicting the doc's stale `2.5.3`).
- `qmd --version` → `2.8.3 (facd35e)`; symlink resolves to a real file inside `node_modules`, not `projects/qmd`.
- `ps -o lstart` on the daemon → start `18:15:24`, running `dist/cli/qmd.js` via node — confirms published code, not the pre-switch tsx/src process.
- `qmd doctor` → **fully clean**: `Runtime: better-sqlite3`, SQLite 3.53.4, sqlite-vec v0.1.9, 63 collections, 3 valid GGUF models downloaded, GPU Metal probe on M3, embedding freshness OK (4,305 docs, fingerprint `c37385`), vector sample reproduces stored vectors.
- Live `qmd query "launchagent daemon management" -c qmd` → query expansion + embeddings + reranking all ran, returned a 96% result — exercises the full node-llama-cpp pipeline end-to-end.
- `curl` POST to `http://localhost:8181/mcp` (`tools/list`) → `HTTP 200`.

## Summary Statistics

- Package switched: source-mode `npm link` (v2.6.3) → published `@tobilu/qmd@2.8.3`.
- Files modified this session: 1 (`CLAUDE.local.md`, +11/−8).
- Verification checks run: 6 (version/dist-tags, symlink resolution, daemon process identity, `doctor`, live `query`, MCP HTTP probe) — all passed.
- Index: 4,305 docs read without migration breakage; no re-embed/re-index run.

## Discoveries / Handoff Notes

- **Published `latest` is `2.8.3`, not `2.5.3`.** The `CLAUDE.local.md` version notes were stale; tobi released through the 2.8.x line after the unpublished 2.6.3. Corrected in the doc.
- **`--allow-scripts` is mandatory in this environment.** A plain `npm install -g @tobilu/qmd` succeeds but ships a broken LLM layer because npm blocks install scripts by default here. Any future `qmd` update via npm must include the `--allow-scripts=…` flag (recorded in the doc).
- **The nvm timebomb persists.** The LaunchAgent plist still hardcodes `/Users/rymalia/.nvm/versions/node/v24.18.0/bin/qmd`, and the global install lives at that same versioned path. The next Node upgrade that removes the `v24.18.0` dir will break the daemon, exactly as in June/July 2026. Unchanged by this switch.
- **The source checkout is now decoupled.** `/Users/rymalia/projects/qmd` stays at v2.6.3 on `dev`; edits there + `npm run build` no longer affect the running `qmd`. Revert to source mode = `npm rm -g @tobilu/qmd` → `npm run build && npm link` from the repo → restart daemon.

## Current State

- Running `qmd`: published `@tobilu/qmd@2.8.3` (global npm).
- MCP daemon: LaunchAgent `com.qmd.mcp` active, serving `http://localhost:8181/mcp`, running `dist/cli/qmd.js` via node.
- Index: `~/.cache/qmd/index.sqlite`, 4,305 docs, fingerprint `c37385`, intact.
- Git: branch `dev`; `CLAUDE.local.md` modified and uncommitted (only changed file).

## Unfinished Work

- **Commit the `CLAUDE.local.md` update** (a commit message was provided this session).
- **Optional:** decide whether to revert the plugin layer (`known_marketplaces.json`) from the local repo to `github:tobi/qmd`, now that dev is off local source mode.
- **Optional / recurring:** the nvm-path timebomb in the LaunchAgent plist remains unaddressed.
