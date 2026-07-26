---
session_id: 8bde7719-4e84-4f20-8a91-6afb8f34b40a
date: 2026-07-26
time: "10:34 AM PDT – 12:50 PM PDT"
project: qmd
branch: dev
related_pr: 794, 795
---

# Session Summary: Upstream PR Campaign Shipped (#794, #795, #796) + Index Recovery

## Overview

Converted yesterday's plugin-staleness discoveries into a complete upstream campaign: PR #794 (fixes #789, Codex-reviewed and hardened) and PR #795 (fixes #790, deliberately non-disruptive scoping) are both filed and MERGEABLE, a third issue (#796, release-skill doc gap) was discovered and filed along the way, and the `qmd-watch-upstream` cloud routine now monitors all eight watched items. Separately, closed out the `qmd pull` fallout: re-embed verified green by doctor after a daemon bounce.

## How the Session Ran

1. **Issue status check**: #789/#790 still open; sole comment on each is the "PowderAddicts" canned helpdesk auto-reply (association: none) — no genuine maintainer response on any of the five open issues.
2. **#794 prep**: worktree `qmd-pr-789` off `upstream/main` (`e428df7`, unchanged since drafting), 4-file patch applied cleanly, sed-stamp simulated, PR-body numbers aligned with the filed issue (Feb snapshot = metadata 1.1.1 / ~151 lines, not the falsified 2.0.0 / ~126).
3. **Codex review (mode 2, gpt-5.6-terra xhigh)**: verdict FIX — one High (the sed stamp didn't fail closed: `A && B` suppresses `set -e`, and a non-matching sed exits 0) + two Lows. All three fixed: grep guard added and functionally tested both ways, PR-body overclaim reworded. User pushed → **PR #794**.
4. **#795 prep**: user chose the issue's *alternative* scoping (`./skills` + both skills) over its lead fix (`./skills/qmd`) to keep the PR zero-disruption; PR body explicitly bridges that discrepancy against issue #790's opinionated framing. Merge-independence from #794 proven with a real `git merge-file` three-way test. User standardized sizes on ~50 KB (installed payload basis, not `du`-of-source), amended, pushed → **PR #795**, MERGEABLE.
5. **Release-skill question** ("does it need node_modules?") → confirmed the cached 230 MB served no consumer at all → incidentally discovered SKILL.md step 1 references `skills/release/scripts/release-context.sh`, **which never existed in any commit** → drafted and filed **issue #796**.
6. **Index recovery**: after user's `qmd embed --force` finished, bounced the LaunchAgent daemon (old PID from Jul 16 holding the pre-pull model) and ran doctor: vector sample 3/3 reproduces (was 3/3 mismatch), MCP endpoint HTTP 200.
7. **Watch routine** updated three times as items landed; renamed `qmd-watch-753-759-760` → `qmd-watch-upstream`.

## Key Decisions Made

- **Independent PRs, not stacked**: #794 and #795 touch different lines of the same manifest; three-way merge test proved clean combination. Only CHANGELOG conflicts (same-location inserts), flagged in the PR body with an offer to rebase whichever lands second.
- **#795 ships the non-disruptive scoping**: removing the release skill is a product call that belongs to the maintainer; the PR notes the optional further trim instead of making the call. (Supporting insight: the release skill is only meaningful inside a qmd dev checkout anyway.)
- **Codex High finding fixed despite matching house style**: the fail-open `&& mv` shape mirrors the existing jq/awk lines, but the silent-no-match failure mode is unique to sed — the grep guard closes exactly the regression class (#789 recurring silently) the PR exists to prevent.
- **Sizes on the installed-payload basis**: user corrected the assistant — 28 KB was `du` of source; measured installs run ~2× raw bytes, so ~50 KB (both skills) is the honest number. Title/body/changelog all aligned via amend.
- **#796 framed around the "uncommitted local file" hypothesis** (user's hunch): `.gitignore` wouldn't hide the script, so it would sit untracked in tobi's checkout — fix list leads with "just `git add` it."

## Changes Made

| Change | Detail |
|--------|--------|
| **PR #794 filed** (user-run) | `fix/plugin-version-sync` off upstream/main: release.sh stamps plugin version with grep guard, catch-up bump 0.1.0 → 2.6.3, release-skill doc, changelog. Fixes #789 |
| **PR #795 filed** (user-run) | `fix/plugin-payload-scope`: `source: "./skills"`, `skills: ["./qmd","./release"]`, changelog. One clean commit `d79ffb6` after ~50 KB amend. Fixes #790 |
| **Issue #796 filed** (user-run) | release skill step 1 runs a script that never existed in any commit; evidence-first body with `git log --all` receipts and three fix options |
| **release.sh hardening** | sed→grep→mv as separate statements under `set -euo pipefail`; grep guard fails the release if the stamp didn't land; both paths functionally tested |
| **qmd-watch-upstream** | Routine `trig_01WbeXf222N9L9RADtsEBW4p` renamed + expanded to PRs #753/#794/#795 and issues #759/#760/#789/#790/#796; PowderAddicts excluded as noise; on #796 close, checks commits API for release-context.sh landing |
| **Daemon bounce** | Old PIDs 6023/6030 (Jul 16, pre-pull model) killed; LaunchAgent restarted in 2 s (PID 22916); MCP HTTP 200 |
| **drafts/** | Both patches + PR bodies synced to what's on GitHub; new `qmd-issue-release-context-gap.md` |
| **Memory** | `project_pr_watch_routine.md` + MEMORY.md updated for rename/expanded scope |

## Issues & PRs

- **Filed** [tobi/qmd#794](https://github.com/tobi/qmd/pull/794) — plugin version-sync (fixes #789)
- **Filed** [tobi/qmd#795](https://github.com/tobi/qmd/pull/795) — plugin payload scoping (fixes #790), MERGEABLE
- **Filed** [tobi/qmd#796](https://github.com/tobi/qmd/issues/796) — release skill references non-existent `release-context.sh`
- Watched but unchanged: #753 (open, no movement), #759/#760 (no comments), #706

## Testing / Research Performed

- **Patch validation**: `git apply --check` clean against `e428df7`; `bash -n`; jq round-trips; sed stamp simulated with dummy versions.
- **Codex independent review** (69 k tokens, read-only): 1 High + 2 Low, all verified by hand before fixing — including reproducing the `set -e` suppression semantics. First guard test was itself invalidated by the exact `||`-suppresses-`set -e` trap; re-tested in a clean bash process (exit 1 before `mv`).
- **Merge-independence**: `git merge-file` three-way test of both PR diffs — clean, merged result carries scoped source + version 2.6.3. GitHub independently reports #795 MERGEABLE.
- **release-skill dependency audit**: byte-level inventory of `skills/` (19.7 KB raw, 4 files), grep of all invoked scripts for Node/npm usage — confirmed nothing consumes the cached node_modules; `git log --all --follow` proved release-context.sh never existed.
- **Post-embed verification**: doctor green end-to-end (vector sample 3/3 reproduces, 3,838 docs on fingerprint c37385, better-sqlite3, Metal probe OK); daemon restart time postdates the kill; MCP initialize returns HTTP 200.

## Summary Statistics

- 2 PRs + 1 issue filed upstream (user-run); 3 upstream artifacts now under watch alongside 5 prior items
- 1 Codex review pass → 3 findings → 3 fixed (1 High, 2 Low)
- 2 git worktrees created (`qmd-pr-789`, `qmd-pr-790`); 0 commits made by the assistant (all user-run per policy)
- 3 RemoteTrigger updates to the watch routine; 1 rename
- 1 daemon bounce; doctor 3/3 vector sample recovered (from 3/3 mismatch)
- 2 assistant claims corrected by the user (28 KB vs ~50 KB payload basis) or by self-test (flawed guard test)

## Discoveries / Handoff Notes

- **`A && B` under `set -e` does not fail closed** — a failing left-hand command is swallowed, and a non-matching `sed` exits 0 anyway. The release.sh guard pattern (separate statements + grep verification) is the fix shape. Also: wrapping a test subshell in `|| echo` disables `set -e` *inside* it — the first guard test passed spuriously because of the very trap being tested.
- **The plugin cache's 230 MB had zero consumers**: skills are instruction files (commands run in the user's cwd, not the cache), and `mcpServers` resolves `qmd` via PATH. True for both skills.
- **`release-context.sh` never existed in any commit** — `63f3b68` (2026-02-16) wrote the reference into SKILL.md without adding the script; nothing in `.gitignore` would hide it, supporting the uncommitted-local-file hypothesis (#796).
- **Installed plugin payload ≈ 2× raw bytes** (block overhead + installer metadata): 13 KB of qmd-skill source measured 24 KB installed; both skills (19.7 KB raw) project to ~50 KB. Use installed-basis numbers in payload discussions.
- **PowderAddicts** is a misconfigured helpdesk auto-responder subscribed to the repo, not a maintainer — excluded from watch-routine signal.
- **Session scratchpads survive session end** (found last session's artifacts intact), but `drafts/` in-repo is the durable home; everything is synced there.

## Current State

- **Checkout**: `dev` @ `6746d11`; untracked: `drafts/` (4 PR/issue artifacts), this summary; uncommitted: none tracked.
- **Worktrees**: `qmd-pr-789` (branch `fix/plugin-version-sync`, pushed) and `qmd-pr-790` (branch `fix/plugin-payload-scope`, pushed) — keep until PRs resolve, then `git worktree remove`.
- **Daemon**: LaunchAgent, PID 22916 (started 12:37:54 today), port 8181, new embed model loaded — coherent with the re-embedded index.
- **Index**: doctor fully green; embeddings current with the post-pull model binary.
- **Plugin**: local 0.1.3 (scoped source, no stdio MCP) — unaffected by today's work; upstream equivalents now in PR.
- **Watch routine**: `qmd-watch-upstream`, next run 9pm PDT tonight.

## Unfinished Work

- **Commit on dev**: this summary (and the still-pending marketplace.json local-config commit from last session, message drafted then). `drafts/` stays deliberately uncommitted — local working artifacts only; the canonical content lives in the filed PRs/issues.
- **CLAUDE.local.md rewrite** of the three-layer/plugin section (directory marketplace, no plugin MCP server, version-bump loop) — carried over.
- **Cache prune** after next Claude Code restart: `rm -rf ~/.claude/plugins/cache/qmd/qmd/0.1.{0,1,2}` (~250 MB) — carried over if not yet done.
- **When #794/#795 get maintainer movement**: whichever merges second needs the trivial changelog rebase (offered in both PR bodies); if tobi asks for the `./skills/qmd` variant on #795, the amend is a two-line manifest edit.
- **nvm timebomb (standing reminder — do not drop)**: LaunchAgent plist still pins `~/.nvm/versions/node/v24.18.0/...`; next Node upgrade kills the daemon. Fix: repoint at `/Users/rymalia/projects/qmd/bin/qmd`.
- **Optional**: Claude Code-side issues from last session (no update `--force`, no drift detection, dep-install for skills-only plugins) remain unfiled.
