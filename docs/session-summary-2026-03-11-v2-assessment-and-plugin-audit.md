# Session Summary: v2.0 Assessment, Branch Planning & Plugin Architecture Audit

**Date:** 2026-03-11 (afternoon)
**Branch:** `dev`
**QMD version:** v2.0.1 (post-upgrade)
**Participants:** rymalia + Claude (Opus 4.6) + secondary agent (branch assessment author)

---

## Purpose

This session had three phases: (1) recall and assess the state of our QMD fork before pulling upstream v2.0, (2) review and refine a branch assessment document created by a secondary agent, and (3) verify the v2.0 upgrade and audit the plugin/skill/MCP architecture.

This summary is written for the developer partner who manages the QMD install. It captures discoveries about the plugin system that aren't documented elsewhere and that directly affect how the QMD MCP integration works.

## Companion Documents

This is one of three docs produced on 2026-03-11 during the v2.0 upgrade. **Read all three** — they cover different aspects and cross-reference each other.

| Document | What It Covers | Read When |
|----------|---------------|-----------|
| [`branch-assessment-2026-03-11-v2-upgrade-plan.md`](branch-assessment-2026-03-11-v2-upgrade-plan.md) | Pre-upgrade branch state (main, dev, feature/mcp-multi-client), upstream changelog analysis, phased upgrade plan (Phases 0–6), what to keep vs drop | You need to understand the branch topology, the upgrade phases, or what happened to the multi-client feature |
| [`session-summary-2026-03-11-v2-upgrade.md`](session-summary-2026-03-11-v2-upgrade.md) | The upgrade execution — runtime switch from Bun to Node, zod pin, sqlite-vec fix, uncommitted working tree state, LaunchAgent status, remaining phases (4c, 5, 6), handoff notes | You're about to do work on the QMD install — **start here** for environment state warnings |
| **This document** | Post-upgrade assessment, branch doc review corrections, plugin/skill/MCP three-layer architecture, v2.0 MCP verification, upstream issues filed | You need to understand how the plugin/MCP/skill pieces fit together, or what issues we filed |

### Critical Warnings From the Upgrade Session

These are documented in detail in the upgrade session summary but are critical enough to repeat here:

1. **Runtime is Node, not Bun.** Binary at `/Users/rymalia/.nvm/versions/node/v24.12.0/bin/qmd`. **Do not run `bun install`** — it recreates `bun.lock`, which makes the `bin/qmd` wrapper switch back to Bun, re-breaking sqlite-vec. Use `npm install` / `npm run build` / `npm link` for all operations.

2. **CLAUDE.local.md is hidden** (renamed to `.bak`). The upstream v2.0 `CLAUDE.md` loads instead, but some of its instructions (like "Use Bun exclusively") are **wrong for our environment**. Phase 5 in the upgrade plan handles cleanup.

3. **LaunchAgent status is ambiguous.** It was stopped (`launchctl bootout`) during the upgrade. It may have been restarted since, but a `vec0` error during this session suggests the daemon may still be running under Bun. Verify with `launchctl list | grep qmd` and check `/tmp/qmd-mcp.error.log`.

4. **Uncommitted changes exist** on `dev`: `package.json` (zod pin), `package-lock.json` (new), `bun.lock` (deleted), plus docs. These should be committed before any branch operations.

---

## What We Did

| Change | Detail |
|--------|--------|
| **Branch state assessment** | Full audit of `main`, `dev` (8 commits), and `feature/mcp-multi-client` (3 commits) against upstream v2.0.1. Confirmed `main` is a clean fast-forward, identified which dev commits to keep vs drop |
| **Branch assessment doc review** | Two rounds of expert review on `docs/branch-assessment-2026-03-11-v2-upgrade-plan.md`. Corrected commit counts, resolved deferred decisions (reward.py → Drop), added "What v2.0 Gives Us" section, fixed MCP tool removal framing, added Node version pre-flight check, added working tree state section |
| **v2.0 MCP verification** | Tested MCP `status`, `query` with `intent` parameter, CLI `--explain` and `--intent` — all working against the upgraded v2.0.1 server |
| **Plugin/skill/MCP architecture audit** | Deep investigation into how the QMD plugin, skill file, and MCP server relate. Discovered the three-layer architecture and version mismatch |
| **`qmd skill install` assessment** | Evaluated the new v2.0.1 command. Determined it's redundant for our setup — the plugin already bundles the same skill file |

---

## Key Finding: The Three-Layer QMD Integration Architecture

This is the most important discovery of the session. The QMD integration in Claude Code is **not one thing** — it's three independent systems that happen to work together:

### Layer 1: MCP HTTP Connection (`~/.claude/.mcp.json`)

```json
{
  "mcpServers": {
    "qmd": {
      "url": "http://localhost:8181/mcp"
    }
  }
}
```

**What it provides:** The actual MCP tools — `query`, `get`, `multi_get`, `status`. These are callable functions that Claude Code invokes during conversations.

**How it works:** Connects to the QMD HTTP daemon running on port 8181 (managed by the LaunchAgent). This is the runtime connection — if the daemon is down, these tools are unavailable.

**Version coupling:** Reflects whatever version the daemon is running. After our upgrade, this serves v2.0.1 tools with the new `intent` parameter, `collections` array, etc.

### Layer 2: Plugin (`~/.claude/plugins/marketplaces/qmd/`)

**What it provides:** The skill file (SKILL.md), metadata, and plugin lifecycle management. The skill is a markdown document that teaches Claude Code *how to write good QMD queries* — syntax reference, combination strategies, examples.

**How it works:** Claude Code clones from `github:tobi/qmd` on every session start (configured in `~/.claude/plugins/known_marketplaces.json`). The plugin cache lives at `~/.claude/plugins/cache/qmd/qmd/0.1.0/`.

**Critical detail — the marketplace.json schema:** The plugin's `marketplace.json` only supports **stdio** transport (`"command"` / `"args"` fields). It does **not** support HTTP `"url"` fields. This was discovered the hard way during the Feb 24 MCP HTTP migration session (see `session-summary-2026-02-24-mcp-http-migration.md`, "The marketplace.json Saga"). This is why the plugin and MCP connection are separate configs — they *must* be, because the plugin schema can't express an HTTP endpoint.

**Version coupling:** The plugin version (`0.1.0`) is independent of the QMD package version (`2.0.1`). The plugin version tracks the *plugin package format*, not QMD itself. The SKILL.md inside says `version: "2.0.0"` in its metadata, and since the plugin re-clones from GitHub on session start, the skill content stays current with upstream main.

### Layer 3: `qmd skill install` (new in v2.0.1)

**What it provides:** A standalone copy of the same SKILL.md, installed to `./.agents/skills/qmd` (project-local) or `~/.agents/skills/qmd` (global).

**Who it's for:** Users who don't use the Claude Code plugin system — maybe they connected QMD as an MCP server directly via `settings.json`, or they use a different MCP client entirely. The skill file gives their LLM the query-writing reference that the plugin would otherwise provide.

**For us: redundant.** We already get the skill via the plugin. Installing it via `qmd skill install` would create a duplicate at a different path. No conflict, but no benefit either.

### How They Compose

```
Plugin (Layer 2)                    MCP Connection (Layer 1)
  │                                    │
  ├── SKILL.md (query reference)       ├── query tool
  ├── marketplace.json (stdio only)    ├── get tool
  └── re-cloned from GitHub            ├── multi_get tool
      on every session start           ├── status tool
                                       └── connects to daemon on :8181

qmd skill install (Layer 3)
  │
  └── Same SKILL.md, standalone copy
      (for non-plugin users)
```

The plugin provides the *knowledge* (how to query). The MCP connection provides the *tools* (the actual query function). They're complementary, not redundant — but they're configured in completely different files and managed by different systems.

### Why This Matters for Maintenance

1. **If the daemon goes down**, MCP tools disappear but the skill still loads (Claude Code can still see the query syntax reference, it just can't execute queries).

2. **If the plugin cache is stale**, the skill might reference old tool names or missing parameters — but the MCP tools still work because they self-describe via the MCP protocol. The risk is the skill teaching the LLM to use a parameter name that changed.

3. **Plugin version `0.1.0` ≠ broken.** The version number is misleading. The content is current because of the GitHub re-clone on session start. Don't chase a "plugin upgrade" — there isn't one to chase. The plugin format version and the QMD software version are different things.

4. **`/plugin` showing "qmd Plugin · qmd · ✔ enabled" and `/mcp` showing "plugin:qmd:qmd · ✔ connected" are two different systems reporting independently.** The plugin can be enabled while the MCP is disconnected (daemon down), or vice versa.

5. **After upgrading QMD, you only need to restart the daemon** (LaunchAgent bootout/bootstrap). The plugin auto-updates from GitHub. The `.mcp.json` config doesn't change. No need to reinstall the plugin or run `qmd skill install`.

---

## Branch Assessment Review: What We Corrected

The branch assessment doc (`docs/branch-assessment-2026-03-11-v2-upgrade-plan.md`) was authored by a secondary agent and went through two rounds of review. Key corrections made:

| Issue | Original | Corrected |
|-------|----------|-----------|
| `finetune/reward.py` line count | "modified" (no numbers) | `+22/-47` — net deletion, simplified entity detection |
| `finetune/reward.py` verdict | "Assess" (deferred) | **Drop** — upstream `4511b9b` went opposite direction (tightened entity detection, added filler penalty vs our simplifications) |
| MCP tool removal framing | Listed under "Breaking Changes in v2.0.0" | Clarified: already removed as of v1.1.0 (our current version at the time), not a new break |
| `collection` param claim | "CLAUDE.md references singular `collection` param" | Replaced with specific skill file references — CLAUDE.md only references CLI flag `-c`, not the MCP param |
| Missing section | No mention of what we gain | Added "What v2.0 Gives Us" table: SDK mode, intent, explain, collection ignore, skill install |
| `bun.lock` framing | "net -34" (small diff) | Added note: will be entirely regenerated, not a cherry-pick candidate |
| Phase 4 install steps | Basic `bun install && build && link` | Added Node version pre-flight check (LaunchAgent plist has hardcoded Node path) |
| Working tree state | Only untracked files listed | Split into untracked + modified, added CHANGELOG.md conflict risk, self-referential note about the assessment doc itself |

---

## v2.0 MCP Verification Results

All tests run against the live v2.0.1 MCP server:

| Test | Status | Notes |
|------|--------|-------|
| `status` | Working | 2,393 docs, 33 collections, all embeddings current |
| `query` with `intent` | Working | "document adapter kuato" with intent correctly returned the three most relevant docs from the documentor collection |
| CLI `--explain` | Working | Full score traces: FTS/vec backend scores, RRF contributions with rank bonus, reranker blend weights |
| CLI `--intent` | Working | Steered expansion toward SDK-related results when disambiguating |
| `query` (lex only) | Working | BM25 keyword search, phrase matching, negation all functional |

### `--explain` Output Format (for reference)

```
Score:  93%
Explain: fts=[0.9364] vec=[none]
  RRF: total=0.0828 base=0.0328 bonus=0.0500 rank=1
  Blend: 75%*1.0000 + 25%*0.7012 = 0.9253
  Top RRF contributions: fts/lex#1:0.0328
```

This breaks down: which backends contributed (FTS vs vec), how RRF fusion combined them (with 2x weight for the first sub-query), whether the top-rank bonus applied, and the final blend between RRF and reranker scores.

### sqlite-vec Error During Vec Search

One `query` call returned `no such module: vec0` — this suggests the MCP server process may have been started under Bun (from before the runtime switch) and needs a restart under Node. The daemon must be running under Node for sqlite-vec to work. See the earlier upgrade session summary for the full Bun vs Node runtime story.

---

## Interesting Discovery: Session Idle Timeout Feedback

While testing `--explain`, a search for "idle timeout session cleanup" surfaced our own session summary from Feb 24 which contained real-world feedback on the timeout value:

> **Session Idle Timeout:** Set to 5 minutes with 60-second cleanup sweeps. **Observation: This may be too aggressive** — both Claude Code and Gemini CLI sessions expired and had to re-initialize during testing. Consider bumping to 15-30 minutes.

If we pursue the idle-timeout PR to upstream (Phase 6 in the upgrade plan), this feedback should inform the default TTL. 15 minutes with a 60-second sweep interval would be more practical than the 5-minute TTL we originally shipped.

---

## Current State

- **QMD version:** v2.0.1 (`qmd --version` confirmed), running under Node v24.12.0
- **MCP server:** Connected during this session (MCP `status` returned 2,393 docs). However, one `query` call returned `no such module: vec0` — the daemon process may be a stale Bun instance from before the runtime switch. If so, it needs a restart under Node
- **Plugin:** v0.1.0 (plugin format version — **not** the QMD version). Content synced from `github:tobi/qmd` on session start. The SKILL.md inside references v2.0 features correctly
- **Skill:** Bundled in plugin at `~/.claude/plugins/cache/qmd/qmd/0.1.0/skills/qmd/SKILL.md`. Also available via `/qmd` skill in Claude Code. `qmd skill install` is redundant — would create a duplicate
- **Branch:** `dev`, 1 commit ahead of `main` (restored local docs), plus uncommitted changes (see upgrade session summary for full list)
- **LaunchAgent:** Uncertain — was stopped during upgrade, may have been restarted. Check with `launchctl list | grep qmd`

### Remaining Upgrade Phases

Phases 0–4b are complete. The following remain (detailed in the companion docs):

| Phase | Description | Documented In |
|-------|-------------|---------------|
| **4c** | Assess v2.0 MCP source, test LaunchAgent manually, re-bootstrap | Upgrade session summary, "Remaining Work" section |
| **5** | CLAUDE.md / CLAUDE.local.md deduplication — strip overlap, update for v2.0 structure | Branch assessment doc, Phase 5 |
| **6** | Evaluate idle timeout as a follow-up PR to upstream — must target new `src/mcp/server.ts`, use 15-min TTL based on real-world feedback | Branch assessment doc, Phase 6 |

### Upstream Issues

All three are filed and open:

| Issue | Title | Fix Complexity |
|-------|-------|---------------|
| [#379](https://github.com/tobi/qmd/issues/379) | `tsc` fails when zod resolves to 4.3.x | One-character diff — good first PR |
| [#380](https://github.com/tobi/qmd/issues/380) | `qmd cleanup` crashes when sqlite-vec unavailable | Small try/catch guard |
| [#381](https://github.com/tobi/qmd/issues/381) | `bin/qmd` wrapper picks wrong runtime for source builds | Needs design discussion with maintainer |

---

## Recommendations for QMD Install Management

1. **After any QMD upgrade from source:** restart the LaunchAgent (`launchctl bootout` then `bootstrap`). The daemon must be running the same version as the installed binary.

2. **Never run `bun install` in the source checkout.** This recreates `bun.lock`, which makes the `bin/qmd` wrapper pick Bun as the runtime, which breaks sqlite-vec. Use `npm install` / `npm run build` / `npm link` exclusively.

3. **The plugin doesn't need manual updates.** It re-clones from `github:tobi/qmd` on every Claude Code session start. The `0.1.0` version number is the plugin *format* version, not the QMD version.

4. **`qmd skill install` is unnecessary** in our setup. The plugin already provides the identical skill file. Only use this command if switching away from the plugin system.

5. **Monitor for `vec0` errors.** If you see `no such module: vec0`, the daemon is running under Bun. Fix: restart under Node (check that `bun.lock` doesn't exist in the source dir, then `launchctl bootout` / `bootstrap`).

6. **The `.mcp.json` and plugin are independent.** Troubleshoot them separately. `/mcp` status in Claude Code shows the MCP connection; `/plugin` shows the plugin/skill status. One can be working while the other is broken.

---

## Upstream Issues Filed

Three issues were filed against `tobi/qmd` based on problems discovered during the v2.0 source build upgrade. They form a coherent picture of the gap between QMD's published npm package (which works flawlessly) and the contributor/fork workflow (which hits all three in sequence).

### The Chain: #381 → #380

| Issue | Title | State | Severity |
|-------|-------|-------|----------|
| [**#379**](https://github.com/tobi/qmd/issues/379) | `tsc` fails when zod resolves to 4.3.x | Open | High — blocks all source builds with npm |
| [**#380**](https://github.com/tobi/qmd/issues/380) | `qmd cleanup` crashes when sqlite-vec unavailable | Open | Medium — crashes instead of graceful degradation |
| [**#381**](https://github.com/tobi/qmd/issues/381) | `bin/qmd` wrapper picks wrong runtime for source builds | Open | Medium — silent ABI mismatch for contributors |

**#381 causes #380.** The `bin/qmd` wrapper checks for `bun.lock` to decide the runtime. Since `bun.lock` is checked into the repo, source builds that use `npm install` still get routed to Bun — where `bun:sqlite` can't load the sqlite-vec extension, causing the `vec0` crash in #380.

**#379 is independent** but hits the same audience. `package.json` declares `"zod": "^4.2.1"` — the caret allows npm to resolve zod 4.3.6, which has breaking type changes against `@modelcontextprotocol/sdk`. One-character fix: remove the caret.

### Why Published Installs Are Unaffected

The npm package avoids all three issues:
- `bun.lock` is excluded by the `"files"` whitelist → wrapper defaults to node (#381 avoided)
- Pre-compiled `dist/` ships → no tsc build step (#379 avoided)
- Node runtime loads sqlite-vec natively → cleanup works (#380 avoided)

### Our Workarounds (Already Applied)

| Issue | Our Fix |
|-------|---------|
| #379 | `npm install zod@4.2.1 --save-exact` (no caret) |
| #380 | Switched runtime to Node, so sqlite-vec loads — crash no longer reachable |
| #381 | Deleted `bun.lock` from source checkout so wrapper picks node |

### PR Opportunities

- **#379** is the easiest contribution — a one-character diff removing `^` from the zod version. Hard to argue against.
- **#380** could be a small PR adding a try/catch guard to `cleanupOrphanedVectors()`, matching the pattern already used elsewhere in `store.ts`.
- **#381** needs design discussion (check for `package-lock.json` first? `.gitignore` bun.lock? document the workaround?) — better left as an issue for the maintainer to weigh in on.
