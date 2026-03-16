# Session Summary: QMD HTTP MCP Proof-of-Concept — Skills-Only Plugin + Shared Daemon

**Date**: 2026-03-12
**Scope**: QMD plugin architecture, MCP transport migration (stdio → HTTP), Claude Code MCP configuration
**Projects touched**: qmd, documentor, ~/.claude config files

## TL;DR

Proved that the QMD Claude Code plugin can be split into **skills-only** (no subprocess) + **HTTP MCP** (shared daemon), eliminating per-session memory duplication. Discovered that `~/.claude/.mcp.json` is not a valid config location — user-scoped MCP servers belong in `~/.claude.json`. The Feb 24 session's HTTP config likely never worked; the plugin's stdio was always doing the real work.

## Key Decisions Made

| Decision | Rationale |
|----------|-----------|
| **Remove `mcpServers` from plugin marketplace.json** | Plugin should provide skills only, not spawn MCP subprocesses. Other plugins (pipecat-dev, swift-skill) prove `mcpServers` is optional. |
| **Use `~/.claude.json` for HTTP MCP config** | Official docs (code.claude.com/docs/en/mcp.md) specify user-scope MCP goes in `~/.claude.json`, not `~/.claude/.mcp.json`. |
| **Point known_marketplaces.json at local repo** | Prevents GitHub re-clone from overwriting our marketplace.json edits. Also a good dev workflow for iterating on plugin config. |
| **Name the MCP server `qmd`** | With `mcpServers` removed from the plugin, the name `qmd` is free. Confirmed: coexists cleanly with the plugin's skill namespace — no collision. |

## Changes Made

| Change | Detail |
|--------|--------|
| **`~/.claude/plugins/cache/qmd/.../marketplace.json`** | Removed `mcpServers` block — skills-only plugin |
| **`~/.claude/plugins/marketplaces/qmd/.../marketplace.json`** | Same removal |
| **`/Users/rymalia/projects/qmd/.claude-plugin/marketplace.json`** | Same removal (source of truth for sync) |
| **`~/.claude/plugins/known_marketplaces.json`** | Changed qmd source from `github:tobi/qmd` → local `git:///Users/rymalia/projects/qmd` |
| **`~/Library/LaunchAgents/com.qmd.mcp.plist`** | Updated binary path from `~/.bun/bin/qmd` → `~/.nvm/versions/node/v24.12.0/bin/qmd` (v2.0.1) |
| **`~/.claude.json`** | Added `mcpServers.qmd` with `type: http`, `url: http://localhost:8181/mcp` (user scope) |
| **`~/.claude/.mcp.json`** | Deleted — stale file at wrong path, never actually worked |

## Incorrect Assumptions Corrected

| Incorrect Assumption | Reality | Impact |
|---------------------|---------|--------|
| **`~/.claude/.mcp.json` is where HTTP MCP servers are configured** | User-scoped MCP servers go in `~/.claude.json`. Project-scoped go in `.mcp.json` at project root. `~/.claude/.mcp.json` is not a recognized location. | The Feb 24 "HTTP MCP connection" via this file was likely never active — the plugin's stdio `mcpServers` was providing all tools |
| **The plugin's stdio and .mcp.json HTTP could coexist as "two separate systems"** | They can coexist, but the Feb 24 setup had the wrong file path, so only stdio ever worked. With the correct path (`~/.claude.json`), HTTP works and stdio can be removed. | 3 unnecessary stdio subprocesses per session, each loading ~2GB of GGUF models |
| **LaunchAgent plist still uses `~/.bun/bin/qmd`** | QMD switched from Bun to Node/npm during v2.0 upgrade (sqlite-vec incompatibility). Binary is at `~/.nvm/versions/node/v24.12.0/bin/qmd` | LaunchAgent would crash on bootstrap with stale path |
| **`mcpServers` is required in plugin marketplace.json** | It's optional. Multiple plugins (pipecat-dev, programming-swift-skill) ship with only `skills` and no `mcpServers`. | Plugin can be skills-only with zero runtime cost |

## Architecture: Before and After

### Before (stdio, per-session)
```
Claude Code Session 1 → spawns qmd mcp (stdio) → loads GGUF models (~2GB)
Claude Code Session 2 → spawns qmd mcp (stdio) → loads GGUF models (~2GB)
Claude Code Session 3 → spawns qmd mcp (stdio) → loads GGUF models (~2GB)
Gemini CLI           → cannot connect (stdio is 1:1)
```
**Total**: 3 processes, ~6GB model memory, no multi-client

### After (HTTP, shared daemon)
```
LaunchAgent daemon (qmd mcp --http, port 8181) → loads GGUF models (~2GB)
  ├── Claude Code Session 1 → connects via HTTP
  ├── Claude Code Session 2 → connects via HTTP
  ├── Claude Code Session 3 → connects via HTTP
  └── Gemini CLI            → connects via HTTP
```
**Total**: 1 process, ~2GB model memory, unlimited clients

## Verification Results

- **Process count**: New sessions spawn 0 qmd subprocesses (confirmed via `ps aux`)
- **MCP connection**: `qmd-http` shows as connected in `/mcp` with 4 tools (query, get, multi_get, status)
- **Skills preserved**: Plugin still provides `qmd` and `release` skills
- **Daemon health**: `curl localhost:8181/health` returns `{"status":"ok"}`
- **Confirmed**: `qmd` as MCP name coexists with plugin skill namespace — no collision

## Upstream PR Roadmap

This proof-of-concept validates a set of changes that could be contributed back to `tobi/qmd`:

### PR 1: Remove `mcpServers` from plugin (small, clean)

**File**: `.claude-plugin/marketplace.json`
- Remove `mcpServers` block, keep `skills`
- Plugin becomes zero-cost (no subprocess, no model loading)
- Users configure MCP transport separately via `claude mcp add`

### PR 2: Update plugin skill docs (follow-up)

**Files**:
- `skills/qmd/SKILL.md` — Fix `expand` query type error (lines 58-61), update `structured_search` → `query` tool name
- `skills/qmd/references/mcp-setup.md` — Rewrite to document both HTTP and stdio transport options, with `claude mcp add` commands
- Include recommended config for HTTP (LaunchAgent daemon) vs stdio (zero-config)

### PR 3: README plugin/MCP section (docs)

**File**: `README.md`
- Add "Plugin vs MCP Server" section explaining the separation
- Document that plugin provides skills/guidance, MCP connection is user-configured
- Show both transport options with copy-paste commands

## LaunchAgent Ghost State (Recurring Issue)

Hit the same launchd ghost state problem from Feb 24:
- `launchctl bootstrap` fails with "I/O error" because the label is already registered
- `launchctl print gui/501/com.qmd.mcp` reveals stale registration with old binary path
- Fix: `launchctl bootout gui/$(id -u)/com.qmd.mcp` then re-bootstrap

This is the third time this has come up. The pattern: any time the plist is modified, you must bootout-then-bootstrap, never just bootstrap.

## Open Items

1. ~~Test `qmd` as MCP name~~ — **confirmed working**, no namespace collision
2. **Close old sessions** — 3 stale stdio processes will die when old terminals close
3. **known_marketplaces.json points to local repo** — intentional for dev, but needs to be reverted to `github:tobi/qmd` after upstream PR merges (or if we want upstream plugin updates)
4. **Phase 4c MCP assessment** (from v2 upgrade plan) — this session effectively completes it: HTTP daemon is the recommended transport, LaunchAgent is the management layer
5. **Upstream PRs** — ready to draft

## How to Revert

If anything breaks:

```bash
# Restore plugin MCP (re-add mcpServers to marketplace.json)
# In: /Users/rymalia/projects/qmd/.claude-plugin/marketplace.json
# Add back: "mcpServers": {"qmd": {"command": "qmd", "args": ["mcp"]}}

# Restore upstream sync
# In: ~/.claude/plugins/known_marketplaces.json
# Change qmd source back to: {"source": "github", "repo": "tobi/qmd"}

# Remove HTTP MCP
claude mcp remove qmd --scope user

# Stop daemon
launchctl bootout gui/$(id -u)/com.qmd.mcp
```

## Summary Statistics

- 7 config files modified
- 1 stale config file deleted (`~/.claude/.mcp.json`)
- 1 LaunchAgent path corrected and re-bootstrapped
- 3 incorrect assumptions from Feb 24 sessions corrected
- 1 proof-of-concept validated (skills-only plugin + HTTP MCP daemon)
- 0 code changes to qmd source (all config/docs changes)
