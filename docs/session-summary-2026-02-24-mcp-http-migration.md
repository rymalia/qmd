# Session Summary: MCP HTTP Migration & Node Package Manager Deep Dive

**Date**: 2026-02-24
**Duration**: Long session with significant debugging

## TL;DR

Migrated QMD's MCP server from stdio (per-session subprocess) to HTTP (shared daemon on port 8181) for Claude Code. Created a macOS LaunchAgent for auto-start at login. Hit a cascade of Node.js native module ABI issues when attempting to migrate global packages from nvm to Homebrew. Reverted to nvm paths after Homebrew's Node 25 proved incompatible with prebuilt native binaries. Learned hard lessons about `launchctl bootstrap` vs the deprecated `load`/`unload` API.

**Phase 2** (second session): Discovered that Claude Code's plugin `marketplace.json` schema doesn't support `"url"` for HTTP MCP — only `"command"`/`"args"` (stdio). The HTTP connection must be configured separately via `~/.claude/.mcp.json`. Also traced a persistent config revert to the plugin system pulling from the upstream GitHub repo on every session start.

## Key Decisions Made

| Decision | Rationale |
|----------|-----------|
| **Switch from stdio to HTTP MCP** | Single shared daemon saves ~2GB memory (one set of GGUF models vs one per session). Multiple agents (Claude Code + Gemini) can share the same endpoint. |
| **Use nvm's Node 24 for LaunchAgent** | Homebrew has Node 25 (current/unstable) which is ABI-incompatible with prebuilt native modules (better-sqlite3). Node 24 (LTS) works out of the box. |
| **Keep Homebrew node@24 installed** | Future option: install globals under `/opt/homebrew/opt/node@24/bin/` for stable paths. Deferred to a future session. |
| **Use `launchctl bootstrap`/`bootout`** | `launchctl load`/`unload` is deprecated on modern macOS and left ghost state after crash loops. |
| **Use `~/.claude/.mcp.json` for HTTP MCP** | Plugin `marketplace.json` schema only supports stdio (`command`/`args`). HTTP `url` field fails schema validation. `.mcp.json` is the correct mechanism for HTTP MCP connections. |
| **Revert marketplace.json to stdio** | Let the plugin system handle skills/metadata via stdio schema. MCP HTTP connection lives separately in `.mcp.json`. Stops the config from being rejected as invalid. |

## Changes Made

| Change | Detail |
|--------|--------|
| **Plugin config (marketplace.json)** | Initially changed to `url` (HTTP), then reverted to `command`/`args` (stdio) after discovering schema validation rejects `url`. All three copies (project source, marketplace, cache) back to stdio. |
| **`~/.claude/.mcp.json` created** | New file — configures QMD HTTP MCP connection globally via `"url": "http://localhost:8181/mcp"`. This is how Claude Code actually connects to HTTP MCP servers. |
| **LaunchAgent created** | `~/Library/LaunchAgents/com.qmd.mcp.plist` — runs `qmd mcp --http` via nvm's Node 24, `RunAtLoad: true`, `KeepAlive: true`, logs to `/tmp/qmd-mcp.{log,error.log}` |
| **Kuato LaunchAgent removed** | `com.kuato.api.plist` was loaded but failing (exit code 1). Unloaded and deleted per user request. |
| **CLAUDE.md updated** | Added "MCP Server (HTTP mode)" section with endpoint info, LaunchAgent management commands. Corrected to reference `.mcp.json` (not marketplace.json) for HTTP config. |
| **Node 101 doc created** | `docs/node-101-package-managers.md` — reference explaining Node, nvm, npm, npx, pnpm, Bun, their relationships, global package stability, package manager isolation ("parallel dimensions"), and LaunchAgent best practices |

## The Homebrew Detour (What Went Wrong)

### Goal
Migrate global npm packages from nvm (fragile versioned paths) to Homebrew (stable `/opt/homebrew/bin/` path).

### What happened
1. Installed qmd, bird, xc, claude-code-viewer globally via Homebrew's npm (**Node 25.6.1**)
2. `better-sqlite3` native addon failed: prebuilt binary was for Node 24 (MODULE_VERSION 137), Homebrew has Node 25 (MODULE_VERSION 141)
3. `npm rebuild` downloaded the same v24 prebuilt — prebuild-install checks the *running* Node version, which was nvm's v24 (PATH pollution)
4. `node-gyp rebuild` compiled against v24 headers — `npx` resolved to nvm's cache which had v24 headers
5. Installed `node@24` LTS via Homebrew — but it's keg-only, globals didn't land on expected paths
6. LaunchAgent got stuck in ghost state from the crash loop — `launchctl load`/`unload` left stale label

### Root causes (diagnosed by external collaborator)
- **nvm pollutes PATH**: Every interactive shell has nvm's node first. All build tools (npm, npx, node-gyp) resolve to nvm's node regardless of which binary you invoke.
- **False positive test**: `node -e "require('better-sqlite3')"` ran under nvm's Node 24 where the v24 binary loads fine — tested the wrong runtime.
- **Golden rule violated**: "The node that runs the build must be the same node that will run the code at runtime."

### Resolution
Reverted to nvm's Node 24 path in the LaunchAgent. Cleaned up all Homebrew globals. Left `node@24` installed for future attempt.

## Files Modified

| File | Action |
|------|--------|
| `~/.claude/plugins/marketplaces/qmd/.claude-plugin/marketplace.json` | `command`/`args` → `url` |
| `~/.claude/plugins/cache/qmd/qmd/0.1.0/.claude-plugin/marketplace.json` | `command`/`args` → `url` |
| `~/Library/LaunchAgents/com.qmd.mcp.plist` | Created (nvm path) |
| `~/Library/LaunchAgents/com.kuato.api.plist` | Deleted |
| `CLAUDE.md` | Added MCP Server HTTP section |
| `docs/node-101-package-managers.md` | Created — full reference doc |

## The marketplace.json Saga (Phase 2)

After the LaunchAgent was working, a second session helped troubleshoot why new Claude Code sessions couldn't connect to the HTTP MCP server.

### The three-layer config chain

```
GitHub tobi/qmd .claude-plugin/marketplace.json   ← upstream source of truth
     ↓ cloned on every session start
~/.claude/plugins/marketplaces/qmd/                ← overwritten from GitHub
     ↓ cached (but not always re-synced)
~/.claude/plugins/cache/qmd/qmd/0.1.0/            ← what Claude Code reads for plugin loading
```

### What went wrong

1. Phase 1 session put `"url": "http://localhost:8181/mcp"` in the cache copy's marketplace.json
2. New sessions failed to load the QMD plugin — debug logs revealed: `Failed to read cached marketplace qmd: Invalid schema: plugins.0.mcpServers: Invalid input`
3. **The plugin marketplace.json schema only supports `"command"`/`"args"` (stdio).** The `"url"` field is not valid in this schema.
4. Claude Code then fell back to cloning from GitHub, got the stdio config, and the plugin loaded — but as a stdio-only skill provider, not an HTTP MCP server
5. The marketplace copy kept reverting because `known_marketplaces.json` lists QMD's source as `github:tobi/qmd` — Claude Code clones fresh from GitHub on every session start

### The fix

- **`~/.claude/.mcp.json`** handles the HTTP MCP connection (supports `"url"`)
- **`marketplace.json`** handles plugin features (skills, metadata) via stdio (`"command"`/`"args"`)
- These are two separate systems — don't mix them

### The wrong-package-manager twist

The user also realized that their previous night's failed attempt to `npm install && npm link` QMD from source was the same class of problem — QMD is a Bun project, and `npm install` produces cryptic failures that look like unstable code. The fix: `bun install && bun link`.

## Current State

- **QMD HTTP daemon**: Running on port 8181, health check passing
- **LaunchAgent**: Loaded via `launchctl bootstrap`, auto-starts at login, uses nvm Node 24 path
- **Claude Code MCP**: Connected via `~/.claude/.mcp.json` (HTTP) — confirmed working with `/mcp` showing `plugin:qmd:qmd · ✔ connected`
- **Claude Code plugin**: Skills loaded via marketplace.json (stdio schema) — `/plugin` showing `qmd Plugin · qmd · ✔ enabled`
- **Git state**: Detached HEAD at `5233e67` (pre-existing from user's previous session, not caused by this work)

## Unfinished Work / Future Tasks

1. **Reattach HEAD to a branch** — `git checkout main` to get off the detached HEAD state
2. **Homebrew node@24 migration** — deferred. The path forward: install globals under node@24's npm, put `/opt/homebrew/opt/node@24/bin` first in LaunchAgent PATH. Key: must override PATH during `npm install -g` too, or use `npm_config_nodedir`.
3. **Gemini agent setup** — user's Gemini agent needs to be pointed at `http://localhost:8181/mcp`

## Lessons Learned

1. **nvm is for project switching, not global tools**: nvm's versioned paths break LaunchAgents, cron jobs, and anything that needs a stable binary location. Use a fixed Node installation (Homebrew, system) for global CLIs.
2. **Native Node addons are fragile across versions**: `better-sqlite3`, `sqlite-vec`, and any C++ addon compiled for one Node major version won't load on another. Prebuilt binaries make this worse because build tools silently download the wrong version.
3. **`launchctl load`/`unload` is deprecated**: Use `launchctl bootstrap gui/$(id -u) <plist>` and `launchctl bootout gui/$(id -u)/<label>`. The old API leaves ghost state after crash loops.
4. **Test under the right runtime**: When validating native modules, ensure the test uses the *exact same node binary* that will run in production. `which node` might not be what you think.
5. **HTTP MCP vs stdio MCP trade-off**: Stdio is zero-config but wasteful (per-session process + model copies). HTTP is efficient (shared process + models) but requires daemon management. For multi-agent setups, HTTP wins.
6. **Plugin marketplace.json ≠ MCP config**: Claude Code's plugin `marketplace.json` schema only supports `"command"`/`"args"` (stdio). HTTP MCP connections must go in `~/.claude/.mcp.json`. These are two separate systems — the plugin handles skills/metadata, `.mcp.json` handles MCP server connections.
7. **Trace config reverts to source of truth**: When a config file keeps reverting, don't keep re-editing it. Find the upstream source. The marketplace copy was being re-cloned from GitHub on every session start — editing the local copy was futile.
8. **Package managers are parallel dimensions**: npm, bun, and pnpm maintain completely separate registries. Installing with one and checking/updating with another leads to ghost duplicates, wrong-version confusion, and cryptic build failures that look like broken code.
9. **Use the project's package manager**: If CLAUDE.md says "use Bun", `npm install` will give cryptic errors that look like the code is broken. It's not — you're using the wrong tool.

## Summary Statistics

- ~15 debugging iterations on the native module ABI mismatch
- ~5 debugging iterations on the marketplace.json schema validation issue
- 3 LaunchAgent path rewrites (nvm → Homebrew → node@24 → nvm)
- 3 marketplace.json round-trips (stdio → url → stdio, discovering schema limitation)
- 1 new config file created (`~/.claude/.mcp.json`)
- 1 Homebrew formula installed (node@24)
- 4 Homebrew global packages installed then uninstalled
- 1 LaunchAgent deleted (kuato)
- 2 new docs created, extensively updated
- 2 Claude Code sessions collaborating via copy-paste relay

---

## Director's Commentary

*The following annotations are from a second Claude Code session that was monitoring the transcript in real time, providing diagnostic assistance via the user copying messages between terminals.*

### Act 1: The False Confidence (npm rebuild loop)

The first sign of trouble was `gyp info using node@24.12.0 | darwin | arm64` in a build that was supposed to target Homebrew's Node 25. From the outside, the diagnosis was immediate: nvm injects itself at the front of PATH in `.zshrc`, so every build tool — `npm`, `npx`, `node-gyp` — resolves `node` to nvm's copy regardless of which npm binary you invoke. The fix was straightforward: `PATH="/opt/homebrew/bin:$PATH" npm rebuild` or `npm_config_nodedir=...`.

What made this hard to see from the inside was the sheer number of moving parts. When you're deep in the iteration loop — delete build dir, rebuild, reload daemon, check health, read logs — it's easy to miss the *constant* in all your failed attempts. Every approach changed the compilation flags but none changed which `node` was on PATH. The forest-for-the-trees problem is real.

### Act 2: The False Positive

The most instructive moment was when `node -e "require('better-sqlite3')"` returned "OK" and the primary session declared victory — only for the LaunchAgent to crash again. From the outside, this was immediately suspicious: the interactive shell's `node` is nvm's v24, the LaunchAgent runs Homebrew's v25. Of course the v24 binary loads under v24. The test proved nothing.

This is a general debugging principle worth internalizing: **when you have multiple installations of anything, bare commands are ambiguous.** Always use absolute paths when verifying: `/opt/homebrew/bin/node -e "require(...)"`, not `node -e "require(...)"`.

### Act 3: The Pivot

The primary session's decision to install `node@24` via Homebrew was a smart lateral move — instead of fighting the build system, match the runtime to the prebuilt binaries. But it introduced a new problem: Homebrew's `node@24` is keg-only, meaning its binaries live in `/opt/homebrew/opt/node@24/bin/` rather than `/opt/homebrew/bin/`. The global `npm install -g` put packages in `/opt/homebrew/lib/` (the main Homebrew prefix), not the keg — so the LaunchAgent was pointing at `/opt/homebrew/opt/node@24/bin/qmd` which didn't exist.

This is a common Homebrew gotcha: keg-only formulas are intentionally *not* symlinked into the main prefix to avoid conflicts. You have to know to use the full keg path or explicitly `brew link --force`.

### Act 4: The Ghost in the Machine

The final obstacle was the most subtle. After reverting to the nvm path, the LaunchAgent silently refused to load — no errors, no process, no entries in `launchctl list`. From the outside, this had all the hallmarks of stale launchd state: the earlier crash loop with `KeepAlive: true` had caused rapid restarts, and the deprecated `launchctl unload` didn't properly clean up the label registration.

The fix — `launchctl bootout gui/$(id -u)/com.qmd.mcp` followed by `launchctl bootstrap` — is the modern API that Apple has been pushing since macOS 10.10 but that almost nobody uses because every tutorial and Stack Overflow answer still shows `load`/`unload`. This was a satisfying diagnosis because the symptom (silent failure) was completely disconnected from the cause (deprecated API + crash loop residue).

### Act 5: The Config That Wouldn't Stick

After the daemon was running, the next surprise was that new Claude Code sessions couldn't see the QMD MCP tools at all. The debug logs told the story: `Invalid schema: plugins.0.mcpServers: Invalid input`. The plugin system's `marketplace.json` schema simply doesn't support HTTP URLs — only stdio `command`/`args`. Every attempt to put `"url"` in marketplace.json was doomed to fail schema validation.

But there was a second layer of confusion: the marketplace copy kept reverting to stdio even after being manually edited. The culprit was `known_marketplaces.json`, which lists QMD's source as `github:tobi/qmd`. On every session start, Claude Code clones fresh from GitHub and overwrites the local marketplace copy. Editing it locally was fighting an automated sync.

The resolution split the responsibilities cleanly: `marketplace.json` handles plugin features (skills, metadata) via its stdio-only schema, while `~/.claude/.mcp.json` handles the HTTP MCP connection. Two systems, two files, no conflict.

### Act 6: Full Circle

The final reveal was the most satisfying. The user mentioned that the previous night, they'd tried to install QMD from source using `npm install && npm link` — and got cryptic errors they attributed to unstable code. They'd rolled back commit by commit trying to find a "working" version, eventually giving up and reinstalling from the npm registry.

The actual problem: QMD is a Bun project. `npm install` can't resolve a `bun.lock` file and fails in ways that look like dependency bugs. The fix was `bun install && bun link` — the same "wrong package manager" class of problem that colored the entire session. The user had been bitten by it before they even started the HTTP migration, they just didn't have the mental model to recognize it.

This is why the "parallel dimensions" section in the Node 101 doc matters. The error messages from using the wrong package manager are indistinguishable from actual bugs — resolution failures, native module build errors, missing binaries. Without knowing that npm/bun/pnpm are completely isolated registries, the natural conclusion is "the code is broken."

### The Meta-Lesson

This session is a case study in **compounding complexity**. The original task — "change the MCP transport from stdio to HTTP" — was a one-line config change. But each step to "improve" the setup (stable paths → Homebrew → native modules → node-gyp → keg-only → launchd → schema validation → config sync) added a layer that interacted badly with the previous layers. The most valuable debugging skill on display was knowing when to stop going deeper and revert to what worked. The primary session got there ("stop fighting Homebrew, go back to nvm") — it just took a few extra rounds.

Having a second set of eyes on the transcript — without the cognitive load of running the commands — made it possible to spot patterns that were invisible from inside the loop. The observer sees the forest; the operator sees the trees. This is, incidentally, the entire value proposition of code review.

And in the end, the most impactful lesson wasn't about native modules or launchd APIs — it was the simplest one: **npm, bun, and pnpm are parallel dimensions, and using the wrong one produces errors that look like someone else's bug.**
