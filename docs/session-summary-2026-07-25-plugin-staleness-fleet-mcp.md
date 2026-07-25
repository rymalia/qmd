---
session_id: 731e9903-79d8-4bc7-9c91-66702eb02703
date: 2026-07-25
time: "2026-07-24 1:33 PM PDT – 2026-07-25 7:27 AM PDT"
resumed: "2026-07-24 3:03 PM PDT, 2026-07-24 4:20 PM PDT"
project: qmd
branch: dev
---

# Session Summary: dev Catch-Up, Agent-Fleet MCP, and the Plugin Staleness Discovery

## Overview

A health check escalated into a productive sprawl: reconciled the `dev` branch with main (now the permanent working branch), moved the entire agent fleet (Claude Code, Codex, Kimi) onto the shared warm HTTP daemon, and — while critiquing the qmd skill — discovered that **no installed qmd plugin has ever received an update** because the plugin version in `.claude-plugin/marketplace.json` has been `0.1.0` since February. Characterized the bug with three controlled experiments, fixed it locally (plugin now 0.1.3: current skill, no plugin MCP server, 24 KB payload), and launched an upstream campaign: issues #789 and #790 filed, version-sync PR drafted and ready to push.

## How We Got Here (session arc)

1. **Health check**: CLI, LaunchAgent daemon, doctor, and MCP all green on v2.6.3 (e428df7).
2. **v2.6.3 provenance solved**: upstream merged release PR #746 (2026-06-24) bumping 2.5.3 → 2.6.3 on main but never pushed the tag — so npm and GitHub releases still say 2.5.3. We run upstream main exactly; "v2.6.3" is upstream's own unpublished version, not a fork artifact.
3. **dev caught up**: merged main into dev (`d39452d`); sole conflict CHANGELOG.md, resolved in main's favor (dev's entries had landed upstream via #718/#719). dev = main + 25 docs commits, zero source divergence. **User now works on dev permanently** (docs live there).
4. **Fleet MCP migration**: Codex `config.toml` and Kimi `mcp.json` switched from stdio (`command: qmd mcp`) to `url: http://localhost:8181/mcp` — all three agents now share the one warm daemon. Kimi verified live end-to-end. Also added OpenAI's Docs MCP (`https://developers.openai.com/mcp`) to Codex and Claude Code (user scope) and tested it.
5. **`qmd pull` fallout**: user's pull refreshed all three models from HF; the new embeddinggemma binary no longer reproduces stored vectors (doctor: 3/3 sample mismatch, distance ~1.03). `qmd embed --force` required — **still pending**.
6. **Skill audit → staleness discovery**: critiqued the repo's SKILL.md (v2.2.0), then found the *served* skill was a February v2.0.0 snapshot from `~/.claude/plugins/cache/qmd/qmd/0.1.0/` — the plugin version never changed, so the cache never invalidated.
7. **Local fixes via three version bumps**: 0.1.1 (validate the version gate), 0.1.2 (drop the plugin's stdio MCP server after it shadowed the HTTP daemon on restart), 0.1.3 (scope `source` to `./skills/qmd` — 24 KB payload).
8. **Upstream campaign**: user filed #789 (version staleness) and #790 (payload; their canonical-install test revealed `plugin install` runs a full npm dependency install — 230 MB, 9,000+ items). PR fixing #789 drafted, validated, ready to push.

## Key Decisions Made

- **Stay on `dev` permanently** — it holds all session summaries and doc-sprint artifacts; source stays identical to main via periodic `git fetch upstream && git merge upstream/main`.
- **HTTP daemon for every agent** — stdio-per-agent duplicates model VRAM and pays cold loads; one warm LaunchAgent daemon serves Claude Code + Codex + Kimi. Applied the same logic to the plugin: its manifest-declared stdio MCP server was removed locally (0.1.2) because it shadowed the HTTP registration.
- **Plugin marketplace is now directory-type** (re-added via local path): reads the working tree live — no clone, no branch pinning, no commit needed for plugin updates. Iteration loop: edit → bump version → `claude plugin update qmd@qmd` → restart.
- **Split the upstream filing** — version-gate bug (#789) and payload bug (#790) have independent fixes and severities; PR targets #789 only, with the #790 scoping fix offered separately.
- **Evidence-first issue writing**: #789 ships three controlled tests (version bump alone → updates flow; content gutted alone → nothing repairs; stock-install `/reload-plugins` after clone edit → cache refreshes).

## Changes Made

| Change | Detail |
|--------|--------|
| **dev merged with main** | `d39452d` — user-run; CHANGELOG conflict resolved theirs; verified 0 commits behind, zero source diff |
| **CLAUDE.local.md** | Install state → dev @ d39452d; unpublished-v2.6.3 note; update recipe covers both remotes (`f32733e`) |
| **Codex MCP** | `~/.codex/config.toml`: qmd → `url = "http://localhost:8181/mcp"`; `openaiDeveloperDocs` added |
| **Kimi MCP** | `~/.kimi-code/mcp.json`: qmd + openaiDeveloperDocs as `url` entries; live smoke test passed |
| **Claude Code MCP** | `openaiDeveloperDocs` added at user scope (`claude mcp add --transport http`) |
| **Plugin 0.1.1** | Version bump validating the cache gate (commit `6403051`, user-run) |
| **Plugin 0.1.2** | Dropped manifest `mcpServers` (stdio server had shadowed the HTTP daemon after restart) |
| **Plugin 0.1.3** | `source: "./skills/qmd"`, `skills: ["./"]` — payload 233 MB → 24 KB, release skill un-shipped; **applies on next restart** |
| **Memory** | New `feedback_nvm_timebomb_reminders.md` (user: "don't stop reminding me"); MEMORY.md indexed |
| **PR draft (scratchpad)** | 4-file patch vs upstream/main: release.sh stamps plugin version (sed, format-preserving), catch-up bump 0.1.0 → 2.6.3, release SKILL.md + CHANGELOG; body opens `Fixes #789. Related: #790` |

## Issues & PRs

- **Filed** [tobi/qmd#789](https://github.com/tobi/qmd/issues/789) — plugin version stuck at 0.1.0 since February; installed plugins never receive updates. Three-test repro.
- **Filed** [tobi/qmd#790](https://github.com/tobi/qmd/issues/790) — plugin install ships the whole repo *and runs a full npm dependency install*: 230 MB / 9,000+ items on a canonical install, for a 24 KB skill. Verified fix: scope `source` to `./skills/qmd`.
- **Drafted, not yet pushed** — PR fixing #789: patch + body + full command sequence (worktree off upstream/main → apply → commit → push → `gh pr create`) handed to user.

## Testing / Research Performed

- **Daemon/CLI health**: doctor (2× — green in the morning, vector-sample regression after `qmd pull`), version cross-checks, process start-time verification, MCP status via HTTP.
- **Merge verification**: `main...dev` counts, tip-to-tip source diff (empty), CHANGELOG conflict-marker scan, `origin/dev` sync.
- **Fleet verification**: `codex mcp list` (URL servers enabled), `kimi doctor` + live `kimi -p` smoke test through the daemon (4,594 docs), Docs MCP search→fetch flow (works; returns huge payloads — search ~109 KB, fetch ignores `anchor` and returns full pages).
- **Plugin forensics**: md5 comparison across repo / marketplace clone / cache copies; cache-dir dating (Feb 20); `claude plugin details` inventories; jq-vs-sed stamp testing (jq reformats inline arrays — rejected); 24 KB scoped-cache verification with byte-identical skill.
- **Upstream recon**: npm registry vs GitHub releases vs git tags for the 2.5.3/2.6.3 split; `git log -S` provenance for the skill wording fixes (landed via #718/#719 as `7488fe8`).

## Summary Statistics

- 3 commits landed on dev (merge d39452d, CLAUDE.local.md f32733e, plugin bump 6403051 — all user-run)
- 3 agent harnesses migrated to the shared HTTP daemon; 1 new remote MCP server added to 2 of them
- 3 plugin version bumps (0.1.1 → 0.1.3); plugin payload 233 MB → 24 KB; 2 upstream issues filed; 1 PR drafted
- 5 months of plugin staleness diagnosed with 3 controlled experiments (2 run here, 1 by user on a second machine)
- 2 of the assistant's claims falsified by user testing (GitHub installs "have no workaround"; canonical installs "only ~13 MB")
- 1 memory file created (nvm-timebomb standing reminder)

## Discoveries / Handoff Notes

- **Plugin cache is version-gated, content-blind**: `claude plugin update` compares only version strings — it won't deliver changed content and won't repair a gutted cache. Fix must be in the repo (stamp version per release).
- **`claude plugin install` runs a dependency install** when the plugin root has a `package.json` — canonical qmd installs materialize 230 MB / 9,000+ items including native modules. Scoping `source` to a dir without package.json (0.1.3) eliminates both copy and install. (This machine's February cache was 13 MB *without* node_modules — behavior apparently changed since.)
- **Directory-type marketplaces read the working tree live** (registered when `marketplace add` gets a local path); the older git-type registration cloned and pinned to the branch checked out at add-time (was main — a second staleness trap, now moot).
- **The plugin's stdio MCP server shadows a same-named user MCP registration** — after restart the session got `mcp__plugin_qmd_qmd__*` tools from a cold stdio spawn while the warm daemon idled. Manifest `mcpServers` removed locally in 0.1.2.
- **`qmd pull` is not a passive health check**: a model refresh that swaps the embedding binary invalidates every stored vector (fingerprint stays `c37385` — it keys on URI, not binary). Doctor's vector-sample check catches it.
- **Daemon coherence during model swap**: the running daemon holds the *old* embed model in memory, so MCP vector search stays consistent with stored vectors until restart; CLI (loading the new model) is the degraded path. Re-embed **before** bouncing the daemon.
- **Docs MCP (`openaiDeveloperDocs`) returns whole pages** (~85–109 KB; `anchor` param broken — "Anchor not found. Returning full document"). Fine in Claude Code (spill-to-file), potentially expensive in Codex/Kimi contexts.
- **qmd collection grew 21 → 66 docs** because the dev checkout put all session summaries in the working tree — project docs are now searchable via qmd.

## Current State

- **Checkout**: `dev` @ `6403051`; uncommitted: `.claude-plugin/marketplace.json` (0.1.3 scoping + mcpServers removal — commit message drafted in-session) and this summary.
- **Daemon**: LaunchAgent, PID from Jul 16, port 8181, healthy — running pre-pull embed model (coherent with stored vectors until restarted).
- **Plugin**: 0.1.3 installed, **applies on next Claude Code restart** (0.1.2 was never live; session ran 0.1.1 with the stdio-MCP side effect). After restart: `rm -rf ~/.claude/plugins/cache/qmd/qmd/0.1.{0,1,2}` (~250 MB; 0.1.0's SKILL.md was intentionally blanked during testing).
- **Index**: 4,596 docs, embeddings **stale vs new model binary** — `qmd embed --force` pending.
- **Scratchpad artifacts** (session-scoped — copy out if the session ends): `qmd-plugin-version-sync.patch`, `qmd-plugin-version-sync-pr-body.md`, both issue drafts, at `/private/tmp/claude-501/-Users-rymalia-projects-qmd/731e9903-79d8-4bc7-9c91-66702eb02703/scratchpad/`.

## Unfinished Work

- **Push the #789 PR** — full command block delivered (worktree off upstream/main, apply patch, commit, push, `gh pr create --repo tobi/qmd`). Patch validated; body opens `Fixes #789. Related: #790`.
- **Decide #790 delivery** — second commit on the PR, separate PR, or leave as issue (scoping fix is verified locally either way). Release-skill note included; user is fine with tobi keeping it.
- **`qmd embed --force`** then daemon restart (`pkill -f "cli/qmd.[tj]s mcp --http"`), then `qmd doctor` to confirm vector sample green.
- **Restart Claude Code** to apply plugin 0.1.3, then prune caches 0.1.0–0.1.2.
- **Commit** the marketplace.json local config (message drafted) and this summary on dev.
- **nvm timebomb (standing reminder — do not drop)**: LaunchAgent plist still pins `~/.nvm/versions/node/v24.18.0/...`; next Node upgrade kills the daemon again. Fix: repoint at `/Users/rymalia/projects/qmd/bin/qmd`. Memory file now enforces the reminder.
- **Optional**: retarget the `qmd-watch` cloud routine to include #789/#790; rewrite CLAUDE.local.md's three-layer/plugin section (directory marketplace, no plugin MCP, version-bump loop); file the Claude Code-side issues (no update `--force`, no drift detection, dep-install for skills-only plugins, `.gitignore` ignored in snapshots).
