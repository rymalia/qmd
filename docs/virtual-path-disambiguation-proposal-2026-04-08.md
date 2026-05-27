# Virtual Path Disambiguation: How an AI Agent Misinterpreted QMD Search Results

**Date**: 2026-04-08
**Author**: Claude (AI agent), prompted by @rymalia
**Context**: During a routine session, an AI agent (Claude) incorrectly concluded that QMD's index contained stale documents. The root cause was a misinterpretation of the `file` field in search results. This document traces the reasoning failure, identifies the documentation gaps that enabled it, and proposes concrete fixes.

---

## What Happened

1. An MCP `query` call returned a result with `file: "tweet/docs/session-summary-2026-03-04-gemini-skill-evaluation.md"` and score 0.93.
2. The agent needed to verify the file existed on disk.
3. The agent constructed the filesystem path `/Users/rymalia/projects/tweet/docs/session-summary-2026-03-04-gemini-skill-evaluation.md` by treating the `file` field as a relative filesystem path rooted at the workspace.
4. That path didn't exist. Neither did `/Users/rymalia/projects/tweet/` as a directory.
5. The agent called `qmd get` on the same path, which returned the full document content.
6. Conclusion: "QMD is serving content for a file that no longer exists on disk. The index is stale."

**This conclusion was wrong.** The file exists at `/Users/rymalia/projects/tweets/docs/session-summary-2026-03-04-gemini-skill-evaluation.md` (plural `tweets`). The `tweet` collection was created with `--name tweet` pointing to the `tweets/` directory. The `file` field uses `collection_name/relative_path` format, not a filesystem path.

## Why It Happened: Three Reasoning Errors

### Error 1: Treated `file` as a filesystem path

The MCP `query` tool returns a `file` field set to `displayPath`, constructed in SQL as:

```sql
d.collection || '/' || d.path as display_path
```

So `tweet/docs/session-summary-...` is `collection_name + '/' + relative_path`. Nothing in the tool's output or description indicated this was a virtual path rather than a filesystem path.

### Error 2: Assumed collection name = directory name

The collection is named `tweet` but points to `/Users/rymalia/projects/tweets`. The `qmd status` output does show the real path, but the agent didn't call `status` to cross-reference — and even if it had, it would need to mentally map `tweet` to `tweets`.

### Error 3: Didn't use `qmd get` for verification first

Instead of using QMD's own `get` tool to verify the file was live (which resolves the virtual path internally and reads from disk), the agent bypassed QMD and went straight to the filesystem with `ls`. If `get` had been used first and returned content, that would have proved the file was live — but the agent would still not have understood *why* the filesystem path didn't match.

## What the Docs Currently Say (or Don't)

| Surface | What it says about `file` / paths | Gap |
|---------|-----------------------------------|-----|
| **MCP `query` tool description** | Nothing about output format — only documents input schema | No mention that `file` is a virtual path |
| **MCP `get` tool description** | `"File path or docid from search results"` | Implies it's a file path, not a virtual/collection-prefixed path |
| **README Output Format** | `"Path: Collection-relative path (e.g., docs/guide.md)"` | Says "collection-relative" (accurate for the part after the prefix), but actual output includes the collection name as a prefix |
| **README Quick Start** | `qmd get "meetings/2024-01-15.md"` | Uses `collection/path` format but doesn't explain that `meetings` is a collection name, not a directory |
| **`qmd status`** | Shows `name` and `path` for each collection | This is the only place where the name-to-path mapping is visible, but consumers won't call `status` just to interpret search results |
| **Code types** | `displayPath: string; // Short display path` | Comment doesn't clarify it's collection-prefixed |

## Proposed Fixes (Ordered by Impact)

### 1. MCP `query` tool: Document the output schema

The `query` tool description is 2,500+ characters of input guidance but says nothing about the output. Add an output section:

```
## Output fields
- `file` — Virtual path in `collection-name/relative-path` format (NOT a filesystem path).
  Collection names are set via `--name` during `collection add` and may differ from
  directory names. Use this value with the `get` tool to retrieve content.
- `docid` — Short hash ID (e.g., #abc123). Also usable with `get`.
- `score` — Relevance score from 0.0 to 1.0.
- `context` — Collection/path context if configured.
- `snippet` — Excerpt around the best matching region.
```

### 2. MCP `get` tool: Clarify `file` parameter description

**Current:**
> `"File path or docid from search results (e.g., 'pages/meeting.md', '#abc123', or 'pages/meeting.md:100' to start at line 100)"`

**Proposed:**
> `"Virtual path (collection-name/relative-path) or docid (#abc123) from search results. This is NOT a filesystem path — collection names may differ from directory names. Use paths exactly as returned by the query tool."`

### 3. README Output Format section: Add a callout

The README says `"Path: Collection-relative path (e.g., docs/guide.md)"` but the actual output is `collection_name/relative_path`. Add:

> **Note:** Paths in search results use the format `collection-name/relative-path`. The collection name is set when you run `qmd collection add --name` and may differ from the filesystem directory name. Use `qmd status` to see the mapping between collection names and filesystem paths.

### 4. Consider adding `source_path` to MCP query results

If the `query` results included the collection's filesystem base path (or a `source` field alongside `file`), the mapping would be self-contained and no cross-referencing with `status` would be needed:

```json
{
  "docid": "#abc123",
  "file": "tweet/docs/guide.md",
  "source": "/Users/rymalia/projects/tweets/docs/guide.md",
  "score": 0.93
}
```

This is the most complete fix but also the most invasive. The documentation fixes (1-3) are likely sufficient for most consumers.

## Connection to Existing Work

This issue is adjacent to two open PRs:

- **#513** — Adds a `source_path` column to preserve original filesystem paths alongside handleized ones. If merged, this column could power the `source` field proposed in fix #4.
- **#520** — Documents handleized paths in search results. The collection-name vs directory-name issue is a separate layer of indirection on top of `handelize()`, but both contribute to the same class of problem: QMD's internal path representation diverging from filesystem reality.

## Summary

The root cause is a **leaky abstraction**. QMD introduces a virtual path namespace (`collection_name/handleized_path`) but exposes it through a field called `file` — a name that strongly implies "filesystem path" to any consumer. The abstraction works perfectly *within* QMD (pass `file` back to `get` and it resolves correctly), but breaks the moment someone uses it *outside* QMD (e.g., to read the file directly from disk).

The fix isn't to remove the abstraction — it's to name it clearly and document the boundary. The `get` tool parameter description is the single highest-leverage change: that's the bridge between "I found something in search" and "I want to verify it exists."
