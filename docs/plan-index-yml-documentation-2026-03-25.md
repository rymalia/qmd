# QMD `index.yml` Documentation Plan

**Date**: 2026-03-25
**Motivation**: Tobi (maintainer) publicly stated the `index.yml` automatic update feature
is "[under documented.](https://x.com/tobi/status/2036748608129671263)" This plan covers the full `index.yml` configuration system, not just
the update command — because none of it is documented in the README.
**Relation to main sprint**: This is a standalone addendum to
`docs/plan-documentation-sprint-2026-03-17-canonical.md` (Phase 1). These additions should
be incorporated into the same upstream documentation PR or submitted as a companion PR.

---

## The Gap

`index.yml` is qmd's **declarative configuration file** — it defines every collection, its
glob pattern, ignore rules, contexts, update commands, and query inclusion. It has been the
single source of truth for collection config since v0.5.0 (Dec 2025), replacing the old
SQLite-based config.

**The README mentions `index.yml` zero times.** Not the file path, not the schema, not the
fact that it's human-editable. Users learn about collections exclusively through CLI
commands (`qmd collection add`, `qmd context add`) with no awareness that these commands
read/write a YAML file they can edit directly.

This matters because:
- Power users (like @davidgasquez) want to manage collections declaratively and version-control the config
- The `update` field (auto-run commands on `qmd update`) is only settable via CLI or YAML editing — and the CLI command `update-cmd` is itself undocumented in the README
- The `ignore` field has no CLI command at all — it can *only* be set by editing the YAML
- The `example-index.yml` shipped in the repo is unreferenced

---

## Evidence Base

All claims below verified against `upstream/main` source (2026-03-25).

### What `index.yml` contains (from `src/collections.ts` types + `example-index.yml`)

```yaml
# ~/.config/qmd/index.yml

global_context: "Universal context applied to all search results"

collections:
  my-notes:
    path: ~/Documents/Notes           # Absolute path to index
    pattern: "**/*.md"                 # Glob pattern for files to include
    ignore:                            # Glob patterns to exclude (optional)
      - "Sessions/**"
      - "*.tmp"
    update: "git pull --rebase"        # Shell command to run on `qmd update` (optional)
    includeByDefault: true             # Include in queries by default (optional, default: true)
    context:                           # Path-prefix context map (optional)
      "/": "Personal notes vault"
      "/journal/2025": "Daily notes from 2025"
```

### Feature inventory with documentation status

| Feature | Source | CLI command | In README? | In `--help`? |
|---------|--------|-------------|------------|--------------|
| Config file location (`~/.config/qmd/index.yml`) | `collections.ts:102-116` | — | **No** | No |
| `global_context` field | `collections.ts:40` | `qmd context add / "text"` | **No** | No |
| `path` field | `collections.ts:28` | `qmd collection add <path>` | Yes (implicitly) | Yes |
| `pattern` field | `collections.ts:29` | `qmd collection add --mask` | **No** (mask flag undocumented) | Yes |
| `ignore` field | `collections.ts:30` | **None — YAML-only** | **No** | No |
| `update` field | `collections.ts:32` | `qmd collection update-cmd` | **No** | No |
| `includeByDefault` field | `collections.ts:33` | `qmd collection include/exclude` | **No** | No |
| `context` map | `collections.ts:22-23` | `qmd context add` | Yes (CLI usage) | No |
| Named indexes (`--index`) | `collections.ts:89`, `qmd.ts:2331` | `qmd --index <name>` | **No** | Yes |
| `QMD_CONFIG_DIR` env var | `collections.ts:104` | — | **No** | No |
| `XDG_CONFIG_HOME` for config | `collections.ts:109` | — | **No** | No |
| `example-index.yml` | repo root | — | **No** (not referenced) | No |

### The `--pull` flag is dead code

`qmd update --pull` is documented in README (line 809) and `--help` (line 2576), but the
parsed value is **never consumed**. The flag is read into `values.pull` at `qmd.ts:2364`
but `updateCollections()` doesn't accept or check any arguments. This is effectively a
no-op. The *actual* mechanism for running commands before re-indexing is the `update` field
in `index.yml` — which is the feature tobi is calling under documented.

**Recommendation**: Either implement `--pull` (run `git pull` in each collection dir) or
remove it from docs/help. Either way, the `update` field should be prominently documented
as the real mechanism.

### How `update` commands execute (qmd.ts:520-554)

1. `qmd update` iterates through all collections
2. For each collection with an `update` field, spawns `bash -c <command>` in the
   collection's directory (`col.pwd`)
3. Stdout/stderr are captured and printed with indentation
4. **On non-zero exit: the entire `qmd update` run aborts** (`process.exit(exitCode)`)
5. On success: proceeds to re-index that collection

This is the feature David Gasquez is using (with `git -C .. pull --ff-only`) and tobi is
calling under documented.

### How `ignore` patterns work (qmd.ts:1483-1492, store.ts:1078-1101)

Ignore patterns are merged with hardcoded always-excluded dirs (`node_modules`, `.git`,
`.cache`, `vendor`, `dist`, `build`) and passed to `fastGlob`'s `ignore` option. User
patterns are additive — you cannot un-ignore the hardcoded dirs.

There is **no CLI command to set ignore patterns**. The only way is to edit `index.yml`
directly. This makes documenting the YAML file essential — without it, users have no way to
discover or use this feature.

### How `includeByDefault` works (collections.ts:223-225)

Collections with `includeByDefault: false` are excluded from all searches unless explicitly
named with `-c`. The field defaults to `true` when absent. Set via:
- `qmd collection exclude <name>` → sets `includeByDefault: false`
- `qmd collection include <name>` → removes the field (reverts to default)

### Config location precedence

1. `QMD_CONFIG_DIR` env var (highest priority) — uses directory directly
2. `XDG_CONFIG_HOME` env var — appends `/qmd/`
3. `~/.config/qmd/` (default)

Config file is `{dir}/{indexName}.yml` where `indexName` defaults to `"index"`.

---

## Proposed README Additions

### Addition 1: Configuration File section

Place **before** the "Collection Management" section — this establishes that collections
live in a YAML file before showing CLI commands that manipulate it.

```markdown
## Configuration

QMD stores collection definitions in a YAML configuration file:

    ~/.config/qmd/index.yml

Every `qmd collection` and `qmd context` command reads and writes this file. You can
also edit it directly — changes take effect on the next command.

### Example

```yaml
# ~/.config/qmd/index.yml

global_context: "Search for [[WikiWord]] links to find related topics."

collections:
  notes:
    path: ~/Documents/Notes
    pattern: "**/*.md"
    context:
      "/": "Personal knowledge base"
      "/journal/2025": "Daily notes from 2025"

  docs:
    path: ~/work/docs
    pattern: "**/*.{md,mdx,txt}"
    ignore:
      - "drafts/**"
      - "*.tmp"
    update: "git pull --rebase"
    context:
      "/": "Team documentation"

  wiki:
    path: ~/external/wiki
    pattern: "**/*.md"
    update: "git -C . pull --ff-only"
    includeByDefault: false
    context:
      "/": "External wiki (excluded from default searches, use -c wiki)"
```

See [`example-index.yml`](example-index.yml) in this repo for a starter template.

### Collection Fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `path` | string | *(required)* | Absolute path to the directory to index |
| `pattern` | string | `"**/*.md"` | Glob pattern for files to include |
| `ignore` | string[] | — | Glob patterns to exclude (e.g. `["drafts/**"]`) |
| `update` | string | — | Shell command to run before re-indexing (see below) |
| `includeByDefault` | boolean | `true` | Include in searches when no `-c` flag is given |
| `context` | map | — | Path-prefix → description (returned with search results) |

### Top-Level Fields

| Field | Type | Description |
|-------|------|-------------|
| `global_context` | string | Context string prepended to all search results across all collections |

### Automatic Update Commands

Each collection can define an `update` command that runs automatically when you
run `qmd update`:

```yaml
collections:
  wiki:
    path: ~/external/wiki
    update: "git pull --ff-only"
```

    $ qmd update
    [1/3] wiki (**/*.md)
        Running update command: git pull --ff-only
        Already up to date.
    Collection: ~/external/wiki (**/*.md)
    Indexed: 0 new, 2 updated, 340 unchanged, 0 removed

The command runs via `bash -c` in the collection's directory. If it exits non-zero,
the update aborts. Set or clear via CLI:

    qmd collection update-cmd wiki 'git pull --ff-only'
    qmd collection update-cmd wiki                        # clear

### Ignore Patterns

Exclude files from indexing with glob patterns:

```yaml
collections:
  notes:
    path: ~/Notes
    ignore:
      - "Sessions/**"
      - "*.tmp"
      - "archive/2023/**"
```

These are additive with qmd's built-in exclusions (`node_modules`, `.git`, `.cache`,
`vendor`, `dist`, `build`). There is no CLI command for ignore patterns — edit the
YAML directly.

### Config Location

The config directory follows this precedence:

| Priority | Source | Example path |
|----------|--------|-------------|
| 1 | `QMD_CONFIG_DIR` env var | `QMD_CONFIG_DIR=/custom/dir` → `/custom/dir/index.yml` |
| 2 | `XDG_CONFIG_HOME` env var | `XDG_CONFIG_HOME=~/.xdg` → `~/.xdg/qmd/index.yml` |
| 3 | Default | `~/.config/qmd/index.yml` |

### Named Indexes

Run multiple independent indexes with `--index`:

    qmd --index work collection add ~/work/docs --name docs
    qmd --index work query "deployment process"

This uses `~/.config/qmd/work.yml` and `~/.cache/qmd/work.sqlite` instead of the
defaults. Useful for separating personal and work knowledge bases.
```

### Addition 2: Environment Variables table update

Add to the existing Environment Variables section (currently only lists `XDG_CACHE_HOME`):

```markdown
| Variable | Default | Description |
|----------|---------|-------------|
| `XDG_CACHE_HOME` | `~/.cache` | Parent directory for `qmd/index.sqlite` |
| `XDG_CONFIG_HOME` | `~/.config` | Parent directory for `qmd/index.yml` |
| `QMD_CONFIG_DIR` | — | Override config directory entirely (takes precedence over `XDG_CONFIG_HOME`) |
```

### Addition 3: Note about `--pull` flag

If `--pull` remains unimplemented, add a note redirecting users:

```markdown
> **Note**: To run commands (like `git pull`) before re-indexing, use the `update`
> field in `index.yml` or `qmd collection update-cmd`. This runs per-collection
> and works with any shell command, not just git.
```

---

## Proposed `example-index.yml` Overhaul

The current `example-index.yml` shows three nearly identical collections — all using
`pattern: "**/*.md"` and `context` only. It demonstrates the minimum viable config but
none of the interesting functionality: no `update`, no `ignore`, no `includeByDefault`,
no `global_context` explanation beyond a single line, no non-markdown patterns, no
multi-level context hierarchy.

For a file whose entire purpose is to teach users the schema, this is a missed opportunity.
The overhaul should make every field visible and commented.

### Proposed replacement

```yaml
# QMD Collections Configuration
# Location: ~/.config/qmd/index.yml
#
# This file defines all collections and their contexts. Every `qmd collection`
# and `qmd context` command reads and writes this file. You can also edit it
# directly — changes take effect on the next command.
#
# Copy this file to ~/.config/qmd/index.yml and customize it.

# ─── Global Context ──────────────────────────────────────────────────────────
# Applied to ALL search results across ALL collections.
# Use this for instructions or conventions that span your entire knowledge base.
global_context: "If you see a relevant [[WikiWord]], you can search for that WikiWord to get more context."

# ─── Collections ─────────────────────────────────────────────────────────────
collections:

  # ── Basic collection: just a path and pattern ──────────────────────────────
  meetings:
    path: ~/Documents/Meetings
    pattern: "**/*.md"
    context:
      "/": "Meeting notes and summaries"

  # ── Hierarchical context: path-prefix → description ────────────────────────
  # Context is matched by path prefix. More specific prefixes take priority.
  # All matching contexts are included in search results (most general first).
  journals:
    path: ~/Documents/Notes
    pattern: "**/*.md"
    context:
      "/": "Personal notes vault"
      "/journal/2024": "Daily notes from 2024"
      "/journal/2025": "Daily notes from 2025"
      "/recipes": "Cooking recipes and meal planning"

  # ── Auto-update: run a command before re-indexing ──────────────────────────
  # The `update` command runs via `bash -c` in the collection's directory
  # every time you run `qmd update`. If it exits non-zero, the update aborts.
  # Set via CLI: qmd collection update-cmd wiki 'git pull --ff-only'
  wiki:
    path: ~/reference/wiki
    pattern: "**/*.md"
    update: "git pull --ff-only"
    context:
      "/": "External wiki — automatically updated from upstream"

  # ── Ignore patterns: exclude files from indexing ───────────────────────────
  # Additive with built-in exclusions (node_modules, .git, .cache, vendor,
  # dist, build). There is no CLI command for this — edit the YAML directly.
  project-docs:
    path: ~/work/project
    pattern: "**/*.{md,mdx,txt}"        # Non-markdown patterns work too
    ignore:
      - "drafts/**"                      # Skip draft documents
      - "*.tmp"                          # Skip temp files
      - "archive/2023/**"                # Skip old archive
    context:
      "/": "Team project documentation"
      "/api": "API reference and endpoint docs"
      "/guides": "How-to guides and tutorials"

  # ── Exclude from default searches ─────────────────────────────────────────
  # Collections with includeByDefault: false are skipped unless you
  # explicitly name them with `-c`: qmd search -c vendor-docs "auth"
  # Set via CLI: qmd collection exclude vendor-docs
  vendor-docs:
    path: ~/work/vendor
    pattern: "**/*.md"
    includeByDefault: false
    context:
      "/": "Third-party vendor documentation (excluded from default searches)"

  # ── Full example: all fields together ──────────────────────────────────────
  codex:
    path: ~/Documents/Codex
    pattern: "**/*.md"
    ignore:
      - "Sessions/**"
    update: "git stash && git pull --rebase --ff-only && git stash pop"
    includeByDefault: true               # true is the default; shown here for clarity
    context:
      "/": "Thematic collections of important concepts and discussions"
```

### What changed vs current file

| Aspect | Current | Proposed |
|--------|---------|----------|
| Header comments | 5 lines, mentions location and editability | 9 lines, adds CLI relationship and copy instruction |
| `global_context` | Present but unexplained | Commented with purpose |
| `update` field | Absent | Shown on two collections with detailed comments |
| `ignore` field | Absent | Full example with three patterns and inline comments |
| `includeByDefault` | Absent | Shown on one excluded collection and one explicit-true |
| Non-markdown patterns | None | `"**/*.{md,mdx,txt}"` example |
| Inline comments | Minimal | Every feature block has a heading comment explaining the concept |
| Number of collections | 3 (all identical structure) | 6 (each demonstrates different features) |
| Context hierarchy | One collection with 3 entries | Multiple depths shown, plus explanation of matching behavior |

---

## Proposed `docs/SYNTAX.md` Additions

No SYNTAX.md changes needed — `index.yml` is configuration, not query syntax.

---

## Interaction with Canonical Plan (Phase 1)

The canonical plan (`plan-documentation-sprint-2026-03-17-canonical.md`) already includes
several items that overlap or complement this work:

| Canonical Plan Item | Overlap | Resolution |
|---------------------|---------|------------|
| §2: Document `update-cmd` subcommand | Subset — only the CLI command | Keep §2, add the YAML `update` field explanation alongside it |
| §2: Document `include`/`exclude` subcommands | Subset — only CLI | Keep §2, add `includeByDefault` YAML field reference |
| §4: Document embed memory flags | None | No overlap |
| §6: MCP parameter reference | None | No overlap |

**Integration strategy**: The new "Configuration" section should appear **before** the
existing "Collection Management" section in the README. This way, when readers hit
`qmd collection add` they already know it's writing to `index.yml`. The canonical plan's
§2 additions (show, include/exclude, update-cmd) then become "CLI shortcuts for editing
the YAML" — a coherent narrative.

---

## What NOT to include

- **Bug fix for `--pull`**: That's a code change, not docs. File separately or mention in
  the PR body as a discovery.
- **SDK `configPath`/inline config**: Already documented in the SDK section of README
  (lines 192-216). No need to duplicate.
- **`findContextForPath()` vs `store.ts` multi-match divergence**: That's a code
  consistency issue, not a documentation task. File as a separate issue if desired.

---

## Validation checklist (before submitting PR)

- [ ] Verify `~/.config/qmd/index.yml` path is correct on a fresh install
- [ ] Verify `update` command runs in collection dir (not cwd) by testing with `pwd` as the command
- [ ] Verify `ignore` patterns work with a test exclusion
- [ ] Verify `--index` creates separate `.yml` and `.sqlite` files
- [ ] Verify `QMD_CONFIG_DIR` override works
- [ ] Verify `--pull` is still a no-op (or check if it's been fixed in latest upstream)
- [ ] Confirm `example-index.yml` is still in the repo root and accurate
