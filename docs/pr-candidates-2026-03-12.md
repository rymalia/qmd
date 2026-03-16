# QMD Upstream PR Candidates — 2026-03-12

**Purpose**: Handoff document for an agent session to prepare and submit PRs against `tobi/qmd`. Contains the what, why, file-level diffs, and context needed to write each PR without re-discovering anything.

**Repo**: https://github.com/tobi/qmd
**Fork**: rymalia/qmd (local at `/Users/rymalia/projects/qmd/`)

---

## Already Submitted

These are done — listed for context so the agent doesn't duplicate work.

| PR/Issue | Title | Status |
|----------|-------|--------|
| [#379](https://github.com/tobi/qmd/issues/379) | Issue: zod caret range allows 4.3.x, breaks tsc | Open |
| [#382](https://github.com/tobi/qmd/pull/382) | PR: pin zod to exact 4.2.1 | Open, fixes #379 |
| [#383](https://github.com/tobi/qmd/issues/383) | Issue: `validateSemanticQuery` rejects hyphenated words as negation | Open |
| [#384](https://github.com/tobi/qmd/pull/384) | PR: fix regex to allow hyphenated words in vec/hyde queries | Open, fixes #383 |

### Already merged upstream (related prior art)

| PR/Issue | Title | Status |
|----------|-------|--------|
| [#361](https://github.com/tobi/qmd/issues/361) | Issue: Launcher falsely detects Bun when `$BUN_INSTALL` is set | Closed (fixed by #362) |
| [#362](https://github.com/tobi/qmd/pull/362) | PR: Remove `$BUN_INSTALL` check from launcher | **Merged** (commit `ae3604c`, 2026-03-11) |

**Why this matters**: #362 was a **partial fix** to `bin/qmd` runtime detection. It removed the `$BUN_INSTALL` environment variable check (false positive for npm global installs when Bun happens to be on the system). But it left the `bun.lock` / `bun.lockb` file check intact — and that check is the false positive that hits **source builds**, which is our #381. Same bug class, different trigger. tobi merged #362, so he clearly agrees this category of runtime misdetection matters.

### Still open from prior session (issues filed, no PRs yet)

| Issue | Title | Notes |
|-------|-------|-------|
| [#380](https://github.com/tobi/qmd/issues/380) | `qmd cleanup` crashes with sqlite-vec error | Root cause is #381 |
| [#381](https://github.com/tobi/qmd/issues/381) | `bin/qmd` runtime detection picks Bun when `bun.lock` exists | Root cause of #380. Only affects source builds — the repo has `bun.lock` because upstream develops with Bun, but contributors building with Node get routed to Bun anyway. Continuation of the same bug class that #362 partially fixed. |

**Note on #380/#381**: These should be a single PR that fixes #381 (the root cause) and closes #380 (the symptom). Frame it as the **second half of #362** — that PR removed the `$BUN_INSTALL` false positive, this one addresses the `bun.lock` false positive for source builds.

**Current `bin/qmd` after #362 merge:**
```sh
# Line 23 — still triggers for source builds that have bun.lock in the repo
if [ -f "$DIR/bun.lock" ] || [ -f "$DIR/bun.lockb" ]; then
  exec bun "$DIR/dist/cli/qmd.js" "$@"
else
  exec node "$DIR/dist/cli/qmd.js" "$@"
fi
```

**Possible fix strategies** (for the PR author to evaluate):
1. **Check for `bun` on PATH instead of lockfile**: `if command -v bun >/dev/null && [ -f "$DIR/bun.lock" ]` — requires both lockfile AND bun binary. Safer but still uses bun for source builds if bun is installed.
2. **Always use Node for dist/**: The `dist/` directory is compiled JS — it runs on any runtime. Node is the safer default since native modules (sqlite-vec) are more reliably compiled for Node. Only use bun if explicitly requested (e.g., `bun qmd.js` directly).
3. **Check how the script was invoked**: If called via a symlink from `~/.bun/bin/`, use bun. If from `~/.nvm/.../bin/`, use node. This is the most precise signal of installation method.

The approach in #362's discussion and tobi's willingness to merge suggests he's receptive to simplifying the detection logic. Reference #362 explicitly in the PR description.

---

## PR Candidate A: Remove `mcpServers` from Plugin

**Priority**: High — this is the most impactful change, proven by our local POC.

### The Problem

The plugin's `marketplace.json` includes an `mcpServers` block that causes Claude Code to spawn a **stdio subprocess** (`qmd mcp`) for every session. Each subprocess loads ~2GB of GGUF models (embeddinggemma + qwen3-reranker) into memory. Three concurrent sessions = three copies = ~6GB wasted.

This also prevents multi-client setups — stdio is 1:1 (one process per client). Users who want Gemini CLI, Claude Desktop, or other clients to share the same QMD instance must use the HTTP transport, but the plugin forces stdio.

### The Fix

Remove the `mcpServers` key from `.claude-plugin/marketplace.json`. The plugin becomes **skills-only** — it provides query guidance, syntax docs, and the `/release` skill, but does not spawn any process. Users configure their MCP connection separately using `claude mcp add`.

### Proof-of-Concept

We validated this locally on 2026-03-12:
- Removed `mcpServers` from marketplace.json
- Added HTTP MCP via `claude mcp add --transport http --scope user qmd http://localhost:8181/mcp`
- Result: plugin skills load, MCP tools served by shared HTTP daemon, zero stdio subprocesses, `qmd` name works for both plugin and MCP server with no collision

### File Changes

**`.claude-plugin/marketplace.json`** — Remove the `mcpServers` block:

```diff
 {
   "plugins": [{
     "name": "qmd",
     "source": "./",
     "description": "Search and retrieve documents from local markdown files.",
     "version": "0.1.0",
     "skills": ["./skills/"],
-    "mcpServers": {
-      "qmd": {
-        "command": "qmd",
-        "args": ["mcp"]
-      }
-    }
   }]
 }
```

That's it — one block removed. Other plugins (pipecat-dev, programming-swift-skill) already ship as skills-only with no `mcpServers`, so this is a proven pattern in the plugin ecosystem.

### PR Description Guidance

Frame this as giving users **choice over their MCP transport** rather than forcing stdio. Key points:
- Skills-only plugins have zero runtime cost (no process, no model loading)
- Users who want stdio can still add it manually: `claude mcp add --transport stdio qmd -- qmd mcp`
- Users who want HTTP (recommended for multi-client) use: `claude mcp add --transport http qmd http://localhost:8181/mcp`
- The plugin name `qmd` and the MCP server name `qmd` coexist without collision
- Reference the memory savings: single HTTP daemon (~2GB) vs N stdio subprocesses (~2GB each)

### What NOT to include in this PR

Don't bundle the skill doc fixes (PR Candidate B) — keep this PR minimal and easy to review. The marketplace.json change is self-contained and reviewable in isolation.

---

## PR Candidate B: Update Plugin Skill Documentation

**Priority**: Medium — follow-up to PR A. Can be submitted independently but makes most sense after A merges.

### Problems in Current Skill Docs

There are factual errors and stale references in the skill files that predate the v2.0 release:

#### `skills/qmd/SKILL.md`

1. **Lines 58-61: Documents a nonexistent `expand` query type for MCP**
   ```
   **expand (auto-expand)**
   - Use a single-line query (implicit) or `expand: question` on its own line
   - Lets the local LLM generate lex/vec/hyde variations
   - Do not mix `expand:` with other typed lines — it's either a standalone expand query or a full query document
   ```
   `expand` is a CLI-only concept from `qmd query`. The MCP `query` tool only accepts `lex`, `vec`, and `hyde` types. The `expand:` prefix does not exist in the MCP tool schema. This misleads MCP consumers into trying a query type that will fail.

   **Fix**: Remove the `expand` row from the query types section entirely. Add a note that auto-expansion is a CLI feature — MCP users should compose their own lex/vec/hyde sub-queries.

2. **Line 83: References `expand:` again**
   ```
   | Don't know vocabulary | Use a single-line query (implicit `expand:`) or `vec` |
   ```
   **Fix**: Change to just `vec` or `vec` + `hyde`.

3. **Line 9: `allowed-tools` references `mcp__qmd__*`**
   ```
   allowed-tools: Bash(qmd:*), mcp__qmd__*
   ```
   This is fine as-is — the wildcard matches whatever the user names their MCP server. No change needed.

#### `skills/qmd/references/mcp-setup.md`

This file has more significant issues:

1. **Line 13-19: Wrong config location and format for Claude Code**
   ```json
   // Shows ~/.claude/settings.json — WRONG
   { "mcpServers": { "qmd": { "command": "qmd", "args": ["mcp"] } } }
   ```
   Claude Code MCP servers are configured via `claude mcp add` (stores in `~/.claude.json`), not by editing `settings.json`. The shown format is also stdio-only with no mention of HTTP.

   **Fix**: Replace with `claude mcp add` commands for both transports:
   ```bash
   # HTTP (recommended — shared daemon, multi-client)
   claude mcp add --transport http --scope user qmd http://localhost:8181/mcp

   # stdio (simple — per-session process, no daemon needed)
   claude mcp add --transport stdio --scope user qmd -- qmd mcp
   ```

2. **Line 52: Wrong tool name**
   ```
   ### structured_search
   ```
   The actual MCP tool is named `query`, not `structured_search`. This was renamed in v2.0.

   **Fix**: Rename to `query`.

3. **Line 64: Wrong parameter shape**
   ```
   "collection": "optional"
   ```
   The parameter is `collections` (plural) and takes an array, not a string:
   ```json
   "collections": ["docs", "notes"]
   ```

   **Fix**: Update to correct parameter name and type.

4. **Lines 42-48: HTTP mode section is minimal**
   The section exists but doesn't explain the LaunchAgent daemon pattern, health checks, or when to prefer HTTP over stdio.

   **Fix**: Expand with:
   - LaunchAgent setup recommendation for persistent daemon
   - Health check: `curl http://localhost:8181/health`
   - When to use HTTP vs stdio (multi-client? → HTTP. Single agent, zero config? → stdio)

### Approach

This PR should be presented as "update plugin docs for v2.0 compatibility" — the errors predate the v2 release and create confusion for new users following the skill guidance.

---

## PR Candidate C: README Plugin/MCP Section

**Priority**: Low — nice-to-have documentation improvement. Not blocking anything.

### The Problem

The README doesn't explain the relationship between the Claude Code plugin (skills) and the MCP server (tools). Users who install the plugin may not understand that:
- The plugin provides guidance/documentation as skills
- The MCP connection is configured separately
- They have a choice of transport (stdio vs HTTP)
- HTTP is recommended for multi-client setups

### The Fix

Add a section to `README.md` (after the existing "MCP Server" section) that explains the plugin architecture:

```markdown
## Claude Code Plugin

QMD ships a Claude Code plugin that provides search guidance and query composition skills.

### Install the plugin

```bash
claude marketplace add tobi/qmd
claude plugin install qmd@qmd
```

### Configure the MCP connection

The plugin provides skills (documentation, query guidance). The MCP connection — which gives Claude access to the search tools — is configured separately:

**HTTP (recommended for multi-client setups):**
```bash
# Start the daemon
qmd mcp --http --daemon

# Add to Claude Code
claude mcp add --transport http --scope user qmd http://localhost:8181/mcp
```

**stdio (simple, per-session):**
```bash
claude mcp add --transport stdio --scope user qmd -- qmd mcp
```

### Why separate?

The plugin is skills-only — it has zero runtime cost. The MCP transport is your choice:
- **HTTP**: Single daemon, shared across Claude Code + Gemini CLI + any HTTP client. One copy of models in memory.
- **stdio**: Zero config, but spawns a new process (and loads models) per session.
```

### Approach

This is pure documentation — no code changes. Can reference the `skills/qmd/references/mcp-setup.md` for detailed setup. If PR B lands first, this section can be shorter since the reference doc would be accurate.

---

## Cross-Cutting Notes for the Agent

### Git workflow

- **Do not commit directly** — the user's preference is to review commit messages and commit themselves
- Each PR should be on its own branch off `main`
- Branch naming: `fix/remove-plugin-mcpservers`, `docs/update-skill-docs-v2`, `docs/readme-plugin-mcp`
- The user's fork is at `rymalia/qmd`, remote is `origin`

### Testing

- Plugin changes: Can't be unit-tested, but the POC was validated manually (see session summary `docs/session-summary-2026-03-12-qmd-http-mcp-poc.md`)
- Skill doc changes: No tests — these are markdown files
- Run existing tests before submitting: `npx vitest run --reporter=verbose test/`

### Tone for PR descriptions

The upstream maintainer is tobi (Tobi Lütke). Keep PRs concise, well-reasoned, and focused. Don't over-explain — the diffs should speak for themselves. Reference issues where applicable. The user has already submitted clean PRs (#382, #384) so match that style.

### Key references

- **Session summary**: `docs/session-summary-2026-03-12-qmd-http-mcp-poc.md` — full POC validation details
- **Audit doc**: `docs/audit-2026-03-01-session-docs-accuracy.md` — identified the skill doc errors
- **Official Claude Code MCP docs**: https://code.claude.com/docs/en/mcp.md — canonical format for `claude mcp add` and `.mcp.json`
- **Existing PRs for style reference**: #382 (zod pin), #384 (regex fix)

### What we learned that isn't obvious

1. `~/.claude/.mcp.json` is NOT a valid config location — user-scope MCP goes in `~/.claude.json` (or via `claude mcp add --scope user`). The Feb 24 session docs that reference this path are wrong.
2. `mcpServers` in `marketplace.json` is optional — proven by pipecat-dev and programming-swift-skill plugins.
3. Plugin name `qmd` and MCP server name `qmd` coexist without collision when `mcpServers` is removed from the plugin.
4. The plugin system re-clones from the GitHub repo listed in `known_marketplaces.json` on every session start — local edits to cached marketplace.json get overwritten. This is why the marketplace.json change must go upstream to stick.
