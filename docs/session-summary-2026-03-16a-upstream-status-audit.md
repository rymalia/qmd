# Session Summary — 2026-03-16 — Upstream Status Audit

## Purpose

Audited the status of all issues and PRs submitted to `tobi/qmd`, checked local branch hygiene, and pulled local `main` up to date with upstream.

## Key Findings

### Issues Filed (4)

| # | Title | Status | Resolution |
|---|-------|--------|------------|
| #379 | tsc fails when zod resolves to 4.3.x | **Closed** | Fixed by our PR #382 |
| #380 | `qmd cleanup` crashes when sqlite-vec unavailable | **Closed** | Fixed by community PR #389 (not ours) |
| #381 | `bin/qmd` picks wrong runtime for source builds | **Closed** | Fixed by our PR #385 |
| #383 | vec/hyde queries reject hyphenated words | **Open** | PR #384 still pending review |

### Pull Requests (3)

| # | Title | Branch | Status |
|---|-------|--------|--------|
| #382 | Pin zod to exact 4.2.1 | `fix/zod-version-pin` | **Merged** |
| #384 | Allow hyphenated words in vec/hyde queries | `fix/semantic-query-hyphen-validation` | **Open** — no reviews/comments |
| #385 | Prioritize package-lock.json in launcher | `fix/launcher-lockfile-priority` | **Merged** |

### Upstream Activity Since Last Sync

~12 merge commits landed on upstream/main since our last pull, including:

- **#370**: `--no-rerank` CLI flag
- **#371**: WSL path detection fix
- **#377**: macOS Homebrew SQLite support for Bun
- **#389**: Skip vector cleanup when sqlite-vec unavailable (independently fixes our #380)
- **#393**: Truncate oversized text before embedding (prevents GGML crash)
- **#395**: Bound memory during embed
- **#396**: Sync stale bun.lock + drift guard (related to our lockfile issues)
- **#399**: ONNX conversion for Transformers.js deployment

### Local Branch Status

| Branch | Purpose | Action |
|--------|---------|--------|
| `dev` (current) | Active WIP — session timeout, marketplace.json, docs | Keep |
| `fix/zod-version-pin` | PR #382 | Safe to delete (merged) |
| `fix/launcher-lockfile-priority` | PR #385 | Safe to delete (merged) |
| `fix/semantic-query-hyphen-validation` | PR #384 | Keep (open) |
| `feature/mcp-multi-client` | Session timeout work | Keep |
| `feat/finetune-mlx-transition` | Finetune exploration | Keep |

### Pull to Main — Done

- Fast-forwarded local `main` to match `upstream/main` (was ~12 commits behind)
- Local `main` now at `2b8f329` (Merge PR #370) — fully in sync
- All WIP on `dev` branch and untracked files were unaffected

## Notable Discoveries

1. **Issue #380 got defense-in-depth**: Our #385 fixed the root cause (wrong runtime detection), while community PR #389 added a separate guard (skip vector cleanup when sqlite-vec missing). Both merged.
2. **PR #396 addressed lockfile drift**: Related to the lockfile detection issues we flagged — upstream synced `bun.lock` and added drift protection.
3. **PR #384 is the only outstanding submission**: No reviews or comments after 4 days. May need a nudge or rebase onto current main.

## Unfinished Work / Next Steps

- **PR #384**: Monitor for review activity; may need rebase onto updated main if conflicts arise from upstream changes
- **Branch cleanup**: Delete `fix/zod-version-pin` and `fix/launcher-lockfile-priority` locally and on origin
- **PR Candidates A/B/C** from `docs/pr-candidates-2026-03-12.md` remain unstarted (remove mcpServers from plugin, update skill docs, README plugin section)
- **Rebase `dev` onto main**: Now that main is updated, consider rebasing dev to pick up the 12 new upstream commits

## Files Modified

None — this was a read-only audit session.

## Summary Statistics

- Issues audited: 4 (3 closed, 1 open)
- PRs audited: 3 (2 merged, 1 open)
- Upstream commits behind: ~12
- Code changes: 0
